# k6 Results Visualization Reference

## Output Options

### Console Summary (Default)

k6 prints an end-of-test summary to stdout by default. No configuration needed.

### JSON Lines (Real-Time Streaming)

```bash
k6 run --out json=results.json script.js
```

Each line is a JSON object with metric data, timestamps, and tags. Useful for post-processing.

### CSV Output

```bash
k6 run --out csv=results.csv script.js
```

### Web Dashboard (Built-in)

```bash
K6_WEB_DASHBOARD=true k6 run script.js
# Access at http://localhost:5665

# Save an HTML report when the test ends (graphs need duration > 3 × K6_WEB_DASHBOARD_PERIOD, default 10s)
K6_WEB_DASHBOARD=true K6_WEB_DASHBOARD_EXPORT=report.html k6 run script.js
```

Real-time visualization in the browser. Shows charts for all metrics. Other variables: `K6_WEB_DASHBOARD_HOST`, `K6_WEB_DASHBOARD_PORT`, `K6_WEB_DASHBOARD_PERIOD`, `K6_WEB_DASHBOARD_OPEN=true` (open the browser automatically).

### OpenTelemetry (Recommended for Metrics Backends)

```bash
# gRPC exporter (default protocol)
K6_OTEL_GRPC_EXPORTER_ENDPOINT=otel-collector:4317 K6_OTEL_GRPC_EXPORTER_INSECURE=true \
k6 run --out opentelemetry script.js

# HTTP/protobuf exporter with basic auth (auth variables need k6 v2.1.0+)
K6_OTEL_EXPORTER_PROTOCOL=http/protobuf \
K6_OTEL_HTTP_EXPORTER_ENDPOINT=otel.example.com:4318 \
K6_OTEL_HTTP_EXPORTER_USERNAME=user K6_OTEL_HTTP_EXPORTER_PASSWORD=secret \
k6 run --out opentelemetry script.js
```

| Variable | Purpose |
|----------|---------|
| `K6_OTEL_EXPORTER_PROTOCOL` | `grpc` (default) or `http/protobuf`. `v1.x:` `K6_OTEL_EXPORTER_TYPE=grpc\|http` |
| `K6_OTEL_GRPC_EXPORTER_ENDPOINT` / `K6_OTEL_HTTP_EXPORTER_ENDPOINT` | Collector address (`host:port`, no scheme). Defaults `localhost:4317` / `localhost:4318` |
| `K6_OTEL_HTTP_EXPORTER_URL_PATH` | HTTP path, default `/v1/metrics` |
| `K6_OTEL_HEADERS` | `k1=v1,k2=v2` headers for the exporter |
| `K6_OTEL_METRIC_PREFIX`, `K6_OTEL_SERVICE_NAME` | Naming; defaults empty prefix, service `k6` |
| `K6_OTEL_EXPORT_INTERVAL`, `K6_OTEL_FLUSH_INTERVAL` | Defaults `10s` and `1s` |
| `K6_OTEL_TLS_*` | Certificate and verification settings |

Mapping: Counter → Int64Counter, Gauge → Float64Gauge, Trend → Float64Histogram, Rate → one counter named after the metric with a `condition` attribute (`zero` / `nonzero`). `v1.x:` the pair-of-counters format (`<metric>.occurred` / `<metric>.total`) could be re-enabled with `K6_OTEL_SINGLE_COUNTER_FOR_RATE=false`; that variable was removed in v2.0.0.

### Prometheus Remote Write

```bash
K6_PROMETHEUS_RW_SERVER_URL=http://prometheus:9090/api/v1/write \
K6_PROMETHEUS_RW_TREND_STATS=p(95),p(99),min,max \
K6_PROMETHEUS_RW_LABELS=environment=staging,server=runner-1 \
k6 run --out experimental-prometheus-rw script.js
```

| Variable | Purpose |
|----------|---------|
| `K6_PROMETHEUS_RW_SERVER_URL` | Remote write endpoint (default `http://localhost:9090/api/v1/write`) |
| `K6_PROMETHEUS_RW_TREND_STATS` | Which trend stats become series when native histograms are off (default `p(99)`) |
| `K6_PROMETHEUS_RW_LABELS` | Static labels on every series, `k=v,k2=v2` (k6 v2.3.0+). Use it to tell several k6 processes apart |
| `--features native-histograms` | Export trends as Prometheus native histograms (k6 v2.1.0+). `v1.x:` `K6_PROMETHEUS_RW_TREND_AS_NATIVE_HISTOGRAM=true` |
| `K6_PROMETHEUS_RW_USERNAME` / `_PASSWORD`, `_BEARER_TOKEN`, `_SIGV4_*` | Authentication |
| `K6_PROMETHEUS_RW_PUSH_INTERVAL` | Default `5s` |
| `K6_PROMETHEUS_RW_STALE_MARKERS` | Mark series stale at test end |

Series labels from script tags win over static labels with the same name.

### InfluxDB (v1 line protocol)

```bash
k6 run --out influxdb=http://influxdb:8086/k6 script.js
k6 run --out influxdb=https://host/influxdb/k6 script.js   # reverse-proxy path prefix, k6 v2.3.0+
```

The docs recommend OpenTelemetry over this output since k6 v1.7.0. InfluxDB v2 needs the `xk6-output-influxdb` extension.

### Multiple Outputs

```bash
k6 run --out json=results.json --out csv=results.csv script.js
```

### Streaming Large Tests

When every sample goes to an external output, drop the work k6 does at the end:

```bash
k6 run --out opentelemetry --no-thresholds --summary-mode=disabled script.js
```

`v1.x:` `--no-summary` is the equivalent flag (removed in v2.0.0).

## handleSummary() Customization

Override the default end-of-test summary:

```javascript
import { textSummary } from 'https://jslib.k6.io/k6-summary/0.1.0/index.js';

export function handleSummary(data) {
  return {
    // Console output
    stdout: textSummary(data, { indent: ' ', enableColors: true }),
    // JSON file
    'summary.json': JSON.stringify(data, null, 2),
    // Custom HTML report
    'report.html': generateHtmlReport(data),
  };
}

function generateHtmlReport(data) {
  const metrics = data.metrics;
  return `
    <html>
    <body>
      <h1>k6 Test Report</h1>
      <h2>HTTP Request Duration</h2>
      <p>Average: ${metrics.http_req_duration.values.avg.toFixed(2)}ms</p>
      <p>P95: ${metrics.http_req_duration.values['p(95)'].toFixed(2)}ms</p>
      <p>P99: ${metrics.http_req_duration.values['p(99)'].toFixed(2)}ms</p>
      <h2>Error Rate</h2>
      <p>${(metrics.http_req_failed.values.rate * 100).toFixed(2)}%</p>
      <h2>Throughput</h2>
      <p>${metrics.http_reqs.values.rate.toFixed(2)} req/s</p>
    </body>
    </html>
  `;
}
```

`handleSummary()` has a time budget of `handleSummaryTimeout` (default `120s`), configurable through `options.handleSummaryTimeout` or `K6_HANDLE_SUMMARY_TIMEOUT` since k6 v2.2.0. Long tests with heavy summaries hit `handleSummary() execution timed out` otherwise. `v1.x:` not configurable.

### handleSummary Data Structure

```javascript
{
  root_group: {
    name: '',
    path: '',
    groups: [...],
    checks: [...],
  },
  metrics: {
    http_req_duration: {
      type: 'trend',
      contains: 'time',
      values: {
        avg: 131.01,
        min: 116.4,
        med: 127.6,
        max: 196.38,
        'p(90)': 146.21,
        'p(95)': 176.91,
      },
      thresholds: {
        'p(95)<500': { ok: true },
      },
    },
    // ... other metrics
  },
}
```

## Grafana Cloud k6

### Running with Cloud Results

```bash
# One-time interactive setup: saves token + stack to the k6 config file (0600 since v2.1.0)
k6 cloud login

# Or non-interactive (CI). Both variables are required since k6 v2.0.0.
export K6_CLOUD_TOKEN=your-token-here
export K6_CLOUD_STACK_ID=your-stack-id

# Run locally, send results to cloud (logs are streamed too since v2.2.0; --no-cloud-logs opts out)
k6 cloud run --local-execution script.js

# Run entirely in cloud
k6 cloud run script.js

# Browse what exists in the stack (v2.0.0 / v2.1.0 / v2.2.0)
k6 cloud project list --format=json
k6 cloud test list --project-id 12345 --json
k6 cloud load-zone list
```

`v1.x:` only the token is required (k6 falls back to the first stack with a warning); `k6 cloud project|test|load-zone list` do not exist.

Exit codes from `k6 cloud run`: `0` finished, `99` aborted by threshold, `97` aborted by user, system, timeout, or script error (v2.0.0+; `v1.x:` returned `0` for those).

### Cloud Dashboard Features

- Real-time metric visualization
- Performance trend comparison across runs
- Threshold pass/fail tracking
- Team sharing and collaboration
- Historical result storage

## Post-Processing JSON Results

### Using jq

```bash
# Extract all http_req_duration data points
cat results.json | jq 'select(.type=="Point" and .metric=="http_req_duration")'

# Get all failed requests
cat results.json | jq 'select(.type=="Point" and .metric=="http_req_failed" and .data.value==1)'

# Timing breakdown hidden by the compact summary
cat results.json | jq -r 'select(.type=="Point" and (.metric|startswith("http_req_"))) | "\(.metric)\t\(.data.value)"'
```

For logs that must be ordered by a log service, run with `k6 --log-ns-timestamps --log-format=json run script.js` (nanosecond timestamps, k6 v2.3.0+).

### Using Python

```python
import json

with open('results.json') as f:
    for line in f:
        data = json.loads(line)
        if data.get('metric') == 'http_req_duration':
            print(f"Duration: {data['data']['value']}ms")
```
