---
name: analyzing-test-results
description: Use when interpreting k6 test output, analyzing performance metrics, identifying bottlenecks, tuning thresholds, or optimizing k6 test scripts. Use when the user has k6 test results and needs to understand response times, error rates, percentiles (p95, p99), or performance degradation patterns.
---

# Analyzing k6 Test Results

Interpret k6 test output, identify performance bottlenecks, and provide optimization recommendations.

## Analysis Workflow

1. **Review summary output** — Check overall pass/fail, iteration count, VU count
2. **Check thresholds** — Which passed/failed? By how much?
3. **Analyze response times** — Look at p50, p90, p95, p99 distribution
4. **Break down request duration** — Examine blocked, connecting, TLS, waiting, receiving
5. **Identify bottlenecks** — Map timing components to root causes
6. **Suggest optimizations** — Script improvements, threshold tuning, infrastructure changes

## Understanding k6 Output

### End-of-Test Summary

```
     data_received..................: 148 kB  4.9 kB/s
     data_sent......................: 17 kB   564 B/s
     http_req_blocked...............: avg=23.38ms  min=1µs    med=5µs    max=235.44ms p(90)=10µs   p(95)=210.47ms
     http_req_connecting............: avg=11.12ms  min=0s     med=0s     max=117.4ms  p(90)=0s     p(95)=99.38ms
   ✓ http_req_duration..............: avg=131.01ms min=116.4ms med=127.6ms max=196.38ms p(90)=146.21ms p(95)=176.91ms
     http_req_failed................: 0.00%  ✓ 0  ✗ 200
     http_req_receiving.............: avg=5.11ms   min=26µs   med=4.88ms max=14.82ms  p(90)=9.31ms p(95)=10.82ms
     http_req_sending...............: avg=27µs     min=4µs    med=23µs   max=240µs    p(90)=42µs   p(95)=66µs
     http_req_tls_handshaking.......: avg=11.7ms   min=0s     med=0s     max=116.76ms p(90)=0s     p(95)=105.06ms
     http_req_waiting...............: avg=125.86ms min=116.27ms med=122.6ms max=185.5ms p(90)=137.21ms p(95)=170.53ms
     http_reqs......................: 200    6.63/s
     iteration_duration.............: avg=1.16s    min=1.12s  med=1.13s  max=1.43s    p(90)=1.15s  p(95)=1.41s
     iterations.....................: 200    6.63/s
     vus............................: 10     min=10  max=10
     vus_max........................: 10     min=10  max=10
```

### Reading the Output

- **✓** = threshold passed, **✗** = threshold failed
- Each metric shows: `avg`, `min`, `med` (median), `max`, `p(90)`, `p(95)`
- `http_reqs` rate = requests per second (RPS)

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
| `checks` | Rate | Assertion pass rate — should be > 99% |
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

## Results Output Options

### JSON Lines (Real-Time)

```bash
k6 run --out json=results.json script.js
```

### CSV Output

```bash
k6 run --out csv=results.csv script.js
```

### Web Dashboard (Built-in)

```bash
K6_WEB_DASHBOARD=true k6 run script.js
# Dashboard at http://localhost:5665
```

### Grafana Cloud k6

```bash
k6 cloud run --local-execution script.js
# Or set K6_CLOUD_TOKEN environment variable
```

See [reference/visualization.md](reference/visualization.md) for detailed visualization options.

## Related Skills

- For designing test scenarios and thresholds: `/k6:designing-test-scenarios`
- For generating test scripts: `/k6:generating-api-load-tests`
