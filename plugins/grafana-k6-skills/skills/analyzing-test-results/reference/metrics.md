# k6 Metrics Reference

## Built-in HTTP Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `http_reqs` | Counter | Total HTTP requests made |
| `http_req_duration` | Trend | Total request time (sending + waiting + receiving) |
| `http_req_blocked` | Trend | Time blocked before initiating request |
| `http_req_connecting` | Trend | Time establishing TCP connection |
| `http_req_tls_handshaking` | Trend | Time performing TLS handshake |
| `http_req_sending` | Trend | Time sending request body |
| `http_req_waiting` | Trend | Time waiting for server response (TTFB) |
| `http_req_receiving` | Trend | Time receiving response body |
| `http_req_failed` | Rate | Rate of failed requests (non-2xx/3xx or expected status) |

## Execution Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `vus` | Gauge | Current number of active virtual users |
| `vus_max` | Gauge | Maximum possible number of VUs |
| `iterations` | Counter | Total number of completed iterations |
| `iteration_duration` | Trend | Time to complete one full iteration |
| `dropped_iterations` | Counter | Iterations that couldn't start (arrival-rate executors) |
| `data_received` | Counter | Amount of data received (bytes) |
| `data_sent` | Counter | Amount of data sent (bytes) |

## Browser Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `browser_web_vital_lcp` | Trend | Largest Contentful Paint |
| `browser_web_vital_fcp` | Trend | First Contentful Paint |
| `browser_web_vital_cls` | Trend | Cumulative Layout Shift |
| `browser_web_vital_inp` | Trend | Interaction to Next Paint |
| `browser_web_vital_ttfb` | Trend | Time to First Byte |
| `browser_web_vital_fid` | Trend | First Input Delay (deprecated) |
| `browser_data_received` | Counter | Browser data received (bytes) |
| `browser_data_sent` | Counter | Browser data sent (bytes) |
| `browser_http_req_duration` | Trend | Browser HTTP request duration |
| `browser_http_req_failed` | Rate | Browser failed request rate |

## WebSocket Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `ws_connecting` | Trend | WebSocket connection time |
| `ws_session_duration` | Trend | Total session duration |
| `ws_msgs_sent` | Counter | Messages sent |
| `ws_msgs_received` | Counter | Messages received |
| `ws_ping` | Trend | Ping round-trip time |
| `ws_sessions` | Counter | Sessions started |

## gRPC Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `grpc_req_duration` | Trend | gRPC request duration |
| `grpc_streams` | Counter | Streams initiated |
| `grpc_streams_msgs_sent` | Counter | Stream messages sent |
| `grpc_streams_msgs_received` | Counter | Stream messages received |

## Check Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `checks` | Rate | Overall rate of successful checks |

## Group Metrics

| Metric | Type | Description |
|--------|------|-------------|
| `group_duration` | Trend | Time to complete a group |

## Custom Metric Types

### Counter — Cumulative sum

```javascript
import { Counter } from 'k6/metrics';
const errors = new Counter('api_errors');

errors.add(1);                    // Increment by 1
errors.add(5);                    // Increment by 5
// Threshold: count, rate
// 'api_errors': ['count<100']
```

### Gauge — Instantaneous value

```javascript
import { Gauge } from 'k6/metrics';
const queueSize = new Gauge('queue_size');

queueSize.add(42);                // Set value
// Threshold: value
// 'queue_size': ['value<1000']
```

### Rate — Percentage of non-zero values

```javascript
import { Rate } from 'k6/metrics';
const successRate = new Rate('success_rate');

successRate.add(true);            // Success (1)
successRate.add(false);           // Failure (0)
successRate.add(1);               // Success
successRate.add(0);               // Failure
// Threshold: rate
// 'success_rate': ['rate>0.95']
```

### Trend — Statistics over time

```javascript
import { Trend } from 'k6/metrics';
const apiLatency = new Trend('api_latency');

apiLatency.add(150);              // Add sample (ms)
apiLatency.add(200);
// Threshold: avg, min, max, med, p(N)
// 'api_latency': ['p(95)<500', 'avg<200']
```

## Metric Type Aggregation Methods

| Type | Available Methods | Example |
|------|------------------|---------|
| Counter | `count`, `rate` | `count<100`, `rate>10` |
| Gauge | `value` | `value<1000` |
| Rate | `rate` | `rate>0.95` |
| Trend | `avg`, `min`, `max`, `med`, `p(N)` | `p(95)<500`, `avg<200` |
