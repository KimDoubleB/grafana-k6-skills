---
name: generating-api-load-tests
description: Use when creating k6 load test scripts for HTTP REST APIs, gRPC services, or WebSocket connections. Use when the user mentions load testing, performance testing, stress testing, API testing, or k6 script generation for protocol-level endpoints.
---

# Generating API Load Tests with k6

Generate k6 load test scripts for protocol-level APIs. Covers HTTP/REST, gRPC, and WebSocket protocols with authentication, data parameterization, checks, and thresholds.

## Quick Start — HTTP Load Test

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  vus: 10,
  duration: '30s',
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};

export default function () {
  const res = http.get('https://test-api.k6.io/public/crocodiles/');

  check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
  });

  sleep(1);
}
```

## Test Lifecycle

Every k6 script follows a 4-stage lifecycle:

```
1. init     — Runs once per VU. Import modules, load files, define options.
              Cannot make HTTP requests.
2. setup()  — Runs once before VU stage. Prepare test data, authenticate.
              Can make HTTP requests. Return value passed to default() and teardown().
3. default  — Runs repeatedly per VU. The main test logic.
              Each iteration resets cookies and tears down connections.
4. teardown — Runs once after all VUs finish. Cleanup resources.
```

```javascript
import http from 'k6/http';

export const options = { vus: 5, duration: '10s' };

export function setup() {
  const loginRes = http.post('https://api.example.com/login', JSON.stringify({
    username: 'testuser', password: 'testpass',
  }), { headers: { 'Content-Type': 'application/json' } });
  return { token: loginRes.json('token') };
}

export default function (data) {
  http.get('https://api.example.com/items', {
    headers: { Authorization: `Bearer ${data.token}` },
  });
}

export function teardown(data) {
  http.post('https://api.example.com/logout', null, {
    headers: { Authorization: `Bearer ${data.token}` },
  });
}
```

## Protocol-Specific Guides

Choose the protocol matching your API:

- **HTTP/REST API**: See [reference/http.md](reference/http.md) — GET/POST/PUT/DELETE, authentication (Basic, Bearer, OAuth, API Key), CRUD patterns, batch requests, file uploads, correlation, cookie handling, HTML forms
- **gRPC**: See [reference/grpc.md](reference/grpc.md) — Client setup, proto loading, unary/streaming calls, metadata, status codes
- **WebSocket**: See [reference/websocket.md](reference/websocket.md) — Connection lifecycle, message handling, event-driven patterns, binary data

## Data Parameterization

For loading test data from external files and generating dynamic data:

See [reference/data-parameterization.md](reference/data-parameterization.md) — SharedArray, CSV/JSON loading, per-VU data, environment variables, dynamic data generation

## Checks

Checks are assertions that do not stop execution on failure. They track pass/fail rates as metrics.

```javascript
import { check } from 'k6';

check(res, {
  'status is 200': (r) => r.status === 200,
  'body contains expected': (r) => r.body.includes('success'),
  'response is JSON': (r) => r.headers['Content-Type'].includes('application/json'),
  'has required field': (r) => r.json('data.id') !== undefined,
});
```

Combine checks with thresholds to enforce pass/fail:

```javascript
export const options = {
  thresholds: {
    checks: ['rate>0.99'], // 99% of checks must pass
  },
};
```

## Basic Thresholds

```javascript
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'],  // 95th percentile < 500ms
    http_req_failed: ['rate<0.01'],                    // Error rate < 1%
    http_reqs: ['rate>100'],                           // At least 100 RPS
    checks: ['rate>0.99'],                             // 99% checks pass
  },
};
```

## Common Patterns

### Tags for Filtering

```javascript
const res = http.get('https://api.example.com/users', {
  tags: { name: 'GetUsers', type: 'api' },
});
```

### Groups for Organization

```javascript
import { group } from 'k6';

export default function () {
  group('User API', function () {
    http.get('https://api.example.com/users');
    http.get('https://api.example.com/users/1');
  });

  group('Product API', function () {
    http.get('https://api.example.com/products');
  });
}
```

### Sleep Between Iterations

```javascript
import { sleep } from 'k6';

export default function () {
  // test logic...
  sleep(Math.random() * 3 + 1); // Random 1-4 second pause (realistic user think time)
}
```

### Custom Metrics

```javascript
import { Trend, Counter, Rate } from 'k6/metrics';

const apiLatency = new Trend('api_latency');
const apiErrors = new Counter('api_errors');
const apiSuccessRate = new Rate('api_success_rate');

export default function () {
  const res = http.get('https://api.example.com/data');
  apiLatency.add(res.timings.duration);
  apiSuccessRate.add(res.status === 200);
  if (res.status !== 200) apiErrors.add(1);
}
```

## Script Generation Workflow

When generating a k6 test script, follow this process:

1. **Identify the target**: URL, protocol (HTTP/gRPC/WebSocket), endpoints
2. **Determine authentication**: Does the API require auth? What type? Generate setup() for token/session acquisition
3. **Define test data**: What parameters vary? Load from CSV/JSON or generate dynamically
4. **Write the default function**: Main test logic with requests, checks, and sleeps
5. **Configure options**: VUs, duration, thresholds based on test goals
6. **Add organization**: Use groups, tags, and custom metrics as needed

## Related Skills

- For advanced load profile configuration (executors, VU patterns, multi-scenario): `/k6:designing-test-scenarios`
- For interpreting test output and optimizing performance: `/k6:analyzing-test-results`
- To generate tests from existing API source code: `/k6:generating-tests-from-code`
- To generate tests from OpenAPI specifications: `/k6:generating-tests-from-openapi`
