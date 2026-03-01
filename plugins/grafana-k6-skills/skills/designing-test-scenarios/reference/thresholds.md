# k6 Thresholds Reference

## Defining Thresholds

### Short Format (Array of Strings)

```javascript
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    http_req_failed: ['rate<0.01'],
  },
};
```

### Long Format (With Abort Control)

```javascript
export const options = {
  thresholds: {
    http_req_duration: [{
      threshold: 'p(99)<1000',
      abortOnFail: true,
      delayAbortEval: '30s',
    }],
  },
};
```

| Field | Type | Description |
|-------|------|-------------|
| `threshold` | string | Threshold expression |
| `abortOnFail` | boolean | Stop test if threshold fails |
| `delayAbortEval` | string | Wait before evaluating abort (e.g., `"10s"`) |

## Threshold Expression Syntax

```
<aggregation_method> <operator> <value>
```

**Operators:** `<`, `<=`, `>`, `>=`, `==`, `!=`

## Aggregation Methods by Metric Type

| Metric Type | Methods | Example Metrics |
|-------------|---------|-----------------|
| **Counter** | `count`, `rate` | `http_reqs`, `data_received` |
| **Gauge** | `value` | `vus`, `vus_max` |
| **Rate** | `rate` | `http_req_failed`, `checks` |
| **Trend** | `avg`, `min`, `max`, `med`, `p(N)` | `http_req_duration`, `iteration_duration` |

**Percentile syntax:** `p(N)` where N is 0.0 to 100 (e.g., `p(90)`, `p(95)`, `p(99)`, `p(99.9)`)

## Common Threshold Patterns

### Response Time

```javascript
thresholds: {
  http_req_duration: [
    'p(50)<200',     // Median under 200ms
    'p(90)<400',     // 90th percentile under 400ms
    'p(95)<500',     // 95th percentile under 500ms
    'p(99)<1000',    // 99th percentile under 1s
    'avg<300',       // Average under 300ms
    'max<3000',      // Max under 3s
  ],
}
```

### Error Rate

```javascript
thresholds: {
  http_req_failed: ['rate<0.01'],   // Less than 1% errors
  checks: ['rate>0.99'],            // 99% of checks pass
}
```

### Throughput

```javascript
thresholds: {
  http_reqs: ['rate>100'],          // At least 100 RPS
  http_reqs: ['count>10000'],       // At least 10000 total requests
}
```

### Custom Metrics

```javascript
import { Trend, Rate, Counter, Gauge } from 'k6/metrics';

const apiLatency = new Trend('api_latency');
const successRate = new Rate('success_rate');
const errorCount = new Counter('error_count');
const queueSize = new Gauge('queue_size');

export const options = {
  thresholds: {
    api_latency: ['p(95)<500', 'avg<200'],   // Trend
    success_rate: ['rate>0.95'],              // Rate
    error_count: ['count<100'],              // Counter
    queue_size: ['value<1000'],              // Gauge
  },
};
```

## Tag-Based Thresholds

Apply different thresholds to different request groups:

```javascript
export const options = {
  thresholds: {
    // Global threshold
    http_req_duration: ['p(95)<500'],
    // Per-endpoint thresholds
    'http_req_duration{name:GetUsers}': ['p(95)<300'],
    'http_req_duration{name:CreateOrder}': ['p(95)<800'],
    'http_req_duration{name:UploadFile}': ['p(95)<2000'],
    // Per-type thresholds
    'http_req_duration{type:api}': ['p(95)<500'],
    'http_req_duration{type:static}': ['p(95)<100'],
  },
};

export default function () {
  http.get('https://api.example.com/users', {
    tags: { name: 'GetUsers', type: 'api' },
  });
  http.post('https://api.example.com/orders', payload, {
    tags: { name: 'CreateOrder', type: 'api' },
  });
}
```

### Group Duration Thresholds

```javascript
export const options = {
  thresholds: {
    'group_duration{group:::login_flow}': ['avg<3000'],
    'group_duration{group:::checkout}': ['avg<5000'],
  },
};
```

Note: Group names in threshold tags use `:::` as separator.

## Combining Checks with Thresholds

Checks alone do NOT affect test exit status. Use thresholds on check rate:

```javascript
export const options = {
  thresholds: {
    checks: ['rate>0.99'],                          // All checks combined
    'checks{myTag:critical}': ['rate>0.999'],       // Tagged checks
  },
};

export default function () {
  const res = http.get(url);

  // Regular check
  check(res, { 'status 200': (r) => r.status === 200 });

  // Tagged check (higher threshold)
  check(res,
    { 'body valid': (r) => r.body.includes('expected') },
    { myTag: 'critical' }
  );
}
```

## Typical SLA Thresholds

### Web Application

```javascript
thresholds: {
  http_req_duration: ['p(95)<500', 'p(99)<1500'],
  http_req_failed: ['rate<0.01'],
  checks: ['rate>0.99'],
}
```

### API Service

```javascript
thresholds: {
  http_req_duration: ['p(95)<200', 'p(99)<500'],
  http_req_failed: ['rate<0.001'],
  http_reqs: ['rate>500'],
}
```

### Real-Time Application

```javascript
thresholds: {
  http_req_duration: ['p(99)<100'],
  http_req_failed: ['rate<0.0001'],
  ws_session_duration: ['p(95)<30000'],
}
```
