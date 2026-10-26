---
name: analyzing-test-results
description: Use when interpreting k6 test output, analyzing performance metrics, identifying bottlenecks, tuning thresholds, or optimizing k6 test scripts. Use when the user has k6 test results and needs to understand response times, error rates, percentiles (p95, p99), or performance degradation patterns.
---

# Analyzing k6 Test Results

Interpret k6 test output, identify performance bottlenecks, and provide optimization recommendations.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before interpreting output or proposing output configuration, determine which k6 produced it:

1. Run `k6 version` where the test ran (prints `k6 v2.3.0 (...)`), or read the Docker tag, the `setup-k6-action` `k6-version` input, or the k6-operator `spec.runner.image`. The summary layout itself is a hint: a `█ TOTAL RESULTS` block means v1.0.0+; a flat metric list is the legacy format (pre-v1.0 or `--summary-mode=legacy`, removed in v2.0.0).
2. If no version is visible, ask the user. Otherwise assume v2.x and say so in the analysis.
3. Apply the notes below when the target is v1.x.

**Version notes for this skill:**
- `--summary-mode` accepts `compact` (default), `full`, and `disabled`. `legacy` and the `--no-summary` flag were removed in v2.0.0. `v1.x:` both still work but are deprecated.
- `handleSummaryTimeout` / `K6_HANDLE_SUMMARY_TIMEOUT` (default `120s`) needs v2.2.0+. `v1.x:` the 120 s budget is fixed.
- OpenTelemetry output: `K6_OTEL_EXPORTER_PROTOCOL=grpc|http/protobuf` (v2.0.0+). `v1.x:` `K6_OTEL_EXPORTER_TYPE=grpc|http`. Basic auth variables need v2.1.0+.
- Prometheus remote write: `K6_PROMETHEUS_RW_LABELS` needs v2.3.0+; native histograms are enabled with `--features native-histograms` (v2.1.0+). `v1.x:` `K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true`.
- `browser_web_vital_fid` was removed in v2.0.0. `v1.x:` it still appears; ignore it in favor of `browser_web_vital_inp`.
- The REST API (`/v1/status`, `/v1/metrics`) is off by default since v2.0.0; start k6 with `--address localhost:6565` to use it. `v1.x:` on by default.
- `k6 cloud run` needs a stack (`k6 cloud login` or `K6_CLOUD_STACK_ID`) since v2.0.0, and `--local-execution` streams logs to Grafana Cloud since v2.2.0 (`--no-cloud-logs` to opt out).
- Nanosecond log timestamps (`--log-ns-timestamps`) need v2.3.0+.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

## Analysis Workflow

1. **Review summary output** — Check overall pass/fail, iteration count, VU count
2. **Check thresholds** — Which passed/failed? By how much?
3. **Analyze response times** — Look at p50, p90, p95, p99 distribution
4. **Break down request duration** — Examine blocked, connecting, TLS, waiting, receiving
5. **Identify bottlenecks** — Map timing components to root causes
6. **Suggest optimizations** — Script improvements, threshold tuning, infrastructure changes

## Understanding k6 Output

### End-of-Test Summary (compact mode, default)

```
  █ THRESHOLDS

    http_req_duration
    ✓ 'p(95)<1500' p(95)=148.21ms
    ✓ 'p(90)<2000' p(90)=146.88ms

    http_req_failed
    ✓ 'rate<0.01' rate=0.00%


  █ TOTAL RESULTS

    checks_total.......................: 90      13.122179/s
    checks_succeeded...................: 100.00% 90 out of 90
    checks_failed......................: 0.00%   0 out of 90

    ✓ quickpizza is up
    ✓ status is 200

    CUSTOM
    custom_waiting_time................: avg=152.355556 min=120      med=141      max=684      p(90)=147.2    p(95)=148.8

    HTTP
    http_req_duration..................: avg=140.36ms   min=119.08ms med=140.96ms max=154.63ms p(90)=146.88ms p(95)=148.21ms
      { expected_response:true }.......: avg=140.36ms   min=119.08ms med=140.96ms max=154.63ms p(90)=146.88ms p(95)=148.21ms
    http_req_failed....................: 0.00%  0 out of 45
    http_reqs..........................: 45     6.56109/s

    EXECUTION
    iteration_duration.................: avg=152.38ms   min=119.37ms med=141.27ms max=684.62ms p(90)=147.11ms p(95)=148.39ms
    iterations.........................: 45     6.56109/s
    vus................................: 1      min=1       max=1
    vus_max............................: 1      min=1       max=1

    NETWORK
    data_received......................: 519 kB 76 kB/s
    data_sent..........................: 4.9 kB 718 B/s
```

### Reading the Output

- **THRESHOLDS** block: every threshold with ✓ (passed) or ✗ (failed) and the observed value. Only present when thresholds are defined.
- **TOTAL RESULTS**: `checks_total` / `checks_succeeded` / `checks_failed` are display-only breakdowns of the `checks` metric (they cannot be used in thresholds or seen by outputs), followed by each check's pass/fail line.
- **CUSTOM / HTTP / EXECUTION / NETWORK** (and GRPC, WEBSOCKET, BROWSER when used): metrics grouped by origin. Trend lines show `avg`, `min`, `med`, `max`, `p(90)`, `p(95)`.
- Indented `{ tag:value }` lines are sub-metrics created by thresholds with tag filters.
- `http_reqs` rate = requests per second (RPS). `http_req_failed` shows the failure rate and the absolute count.
- Compact mode hides `http_req_blocked`, `http_req_connecting`, `http_req_tls_handshaking`, `http_req_sending`, `http_req_waiting`, `http_req_receiving`. Use `k6 run --summary-mode=full` to see them (plus per-group and per-scenario sections), or request the timing breakdown from JSON output.
- `k6 run --summary-mode=disabled` turns the summary off, for example when streaming to an external output. `v1.x:` `--no-summary` also works; `legacy` prints the flat pre-v1.0 list.

## HTTP Request Duration Breakdown

The total `http_req_duration` is composed of:

```
blocked → connecting → tls_handshaking → sending → waiting → receiving
                                                      ↑
                                              (TTFB / server time)
```

| Component | What It Measures | Typical Cause of High Values |
|-----------|-----------------|------------------------------|
| `http_req_blocked` | Time in queue before request starts | Connection pool exhaustion, DNS lookup |
| `http_req_connecting` | TCP connection time | Network latency, server far away |
| `http_req_tls_handshaking` | TLS/SSL handshake | Missing keep-alive, new connections per request |
| `http_req_sending` | Time to send request body | Large request body, slow upload |
| `http_req_waiting` | Server processing time (TTFB) | Slow backend, DB queries, server overload |
| `http_req_receiving` | Time to download response | Large response body, slow network |
| `http_req_duration` | sending + waiting + receiving | Overall request time |

See [reference/bottleneck-patterns.md](reference/bottleneck-patterns.md) for detailed bottleneck analysis patterns.

## Key Metrics Quick Reference

See [reference/metrics.md](reference/metrics.md) for the complete k6 metrics reference.

### Most Important Metrics to Monitor

| Metric | Type | What to Watch |
|--------|------|---------------|
| `http_req_duration` | Trend | p95 and p99 — are they within SLA? |
| `http_req_failed` | Rate | Error rate — should be < 1% for normal load |
| `http_reqs` | Counter | Request rate (RPS) — is target throughput achieved? |
| `iteration_duration` | Trend | Full iteration time including sleeps |
| `checks` | Rate | Assertion pass rate — should be > 99%. Shown as `checks_total/succeeded/failed` in the summary |
| `vus` | Gauge | Active VU count — did it reach target? |

## Percentile Interpretation

| Percentile | Meaning |
|------------|---------|
| `p(50)` / `med` | Half of requests are faster. Typical user experience. |
| `p(90)` | 90% of requests are faster. Most users' experience. |
| `p(95)` | 95% of requests are faster. Common SLA target. |
| `p(99)` | 99% of requests are faster. Worst-case for most users. |
| `p(99.9)` | 1 in 1000 requests slower. Tail latency. |

**Rule of thumb:** If p95 is 2x+ the median, there's likely a bottleneck affecting some requests.

## handleSummary() Customization

Override the default summary output:

```javascript
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.1.0/index.js';

export function handleSummary(data) {
  return {
    'summary.json': JSON.stringify(data, null, 2),
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
  };
}
```

Available output targets:
- `stdout` — Console output
- `stderr` — Error output
- `'filename.json'` — Write to file
- `'filename.html'` — Write HTML report

`handleSummary()` must finish within `handleSummaryTimeout` (default `120s`; set `handleSummaryTimeout: '5m'` in options or `K6_HANDLE_SUMMARY_TIMEOUT`, k6 v2.2.0+). `v1.x:` the 120 s limit is fixed, so keep the function light.

## Results Output Options

### JSON Lines (Real-Time)

```bash
k6 run --out json=results.json script.js
```

### CSV Output

```bash
k6 run --out csv=results.csv script.js
```

### OpenTelemetry (recommended for metrics backends)

```bash
K6_OTEL_EXPORTER_PROTOCOL=grpc K6_OTEL_GRPC_EXPORTER_ENDPOINT=otel-collector:4317 \
k6 run --out opentelemetry script.js
```

Since k6 v1.7.0 the docs recommend OpenTelemetry over the InfluxDB output. `v1.x:` the protocol variable is `K6_OTEL_EXPORTER_TYPE=grpc|http`.

### Prometheus Remote Write

```bash
K6_PROMETHEUS_RW_SERVER_URL=http://prometheus:9090/api/v1/write \
K6_PROMETHEUS_RW_LABELS=environment=staging,job=checkout \
k6 run --out experimental-prometheus-rw script.js
```

`K6_PROMETHEUS_RW_LABELS` (v2.3.0+) separates several k6 instances writing to one Prometheus. Enable native histograms for trends with `--features native-histograms` (v2.1.0+).

### Web Dashboard (Built-in)

```bash
K6_WEB_DASHBOARD=true k6 run script.js
# Dashboard at http://localhost:5665
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=report.html k6 run script.js   # also save an HTML report
```

### Grafana Cloud k6

```bash
k6 cloud login                      # interactive: token + stack, saved to the k6 config file
k6 cloud run --local-execution script.js   # run locally, stream results (and logs since v2.2.0) to the cloud
k6 cloud run script.js                     # run on cloud load generators

# Non-interactive (CI): both variables are required since v2.0.0
K6_CLOUD_TOKEN=$TOKEN K6_CLOUD_STACK_ID=$STACK k6 cloud run --local-execution --no-cloud-logs script.js
```

`v1.x:` the stack is optional (k6 falls back to the first stack with a warning) and logs are not streamed.

See [reference/visualization.md](reference/visualization.md) for detailed visualization options.

## Related Skills

- For designing test scenarios and thresholds: `/k6:designing-test-scenarios`
- For generating test scripts: `/k6:generating-api-load-tests`
- For the k6 version matrix, CI exit codes, and cloud configuration: `/k6:operating-k6-in-ci-cd`
