---
name: designing-test-scenarios
description: Use when designing k6 load profiles, choosing executors, configuring VU patterns, setting thresholds, or planning test scenarios. Use when the user asks about stress test, spike test, soak test, breakpoint test, load model, executor selection, or pass/fail criteria.
---

# Designing k6 Test Scenarios

Design load profiles, select executors, configure thresholds, and orchestrate multi-scenario tests. This skill helps choose the right approach for your performance testing goals.

## Test Type Selection Guide

Choose a test type based on your goal:

| Goal | Test Type | VUs | Duration | Details |
|------|-----------|-----|----------|---------|
| Verify script works, get baseline | **Smoke** | 2-5 | 30s-3m | Run after every script change |
| Assess typical load performance | **Load** | Average production | 5-60min | Gradual ramp-up, sustained plateau |
| Check behavior under heavy load | **Stress** | 50-100%+ above avg | 10-60min | Higher than normal, expect degradation |
| Detect long-running issues (leaks) | **Soak** | Average production | 3-72 hours | Same as load but extended duration |
| Survive sudden traffic burst | **Spike** | Very high, sudden | 2-5min | Rapid ramp, minimal plateau |
| Find system capacity limits | **Breakpoint** | Incremental to failure | Until break | Continuous ramp, no plateau |

See [reference/test-types.md](reference/test-types.md) for detailed configurations and stage patterns per test type.

## Executor Selection

```
What drives your test?
│
├─ Fixed number of total iterations → shared-iterations
│   (e.g., "run exactly 1000 requests")
│
├─ Fixed iterations per VU → per-vu-iterations
│   (e.g., "each VU runs exactly 50 requests")
│
├─ Constant VU count for duration → constant-vus
│   (e.g., "keep 20 users active for 5 minutes")
│
├─ Variable VU count over time → ramping-vus
│   (e.g., "ramp from 0 to 100 users over 10 minutes")
│
├─ Constant request rate (RPS) → constant-arrival-rate
│   (e.g., "maintain 50 requests/second")
│
└─ Variable request rate over time → ramping-arrival-rate
    (e.g., "ramp from 10 to 100 requests/second")
```

See [reference/executors.md](reference/executors.md) for complete configuration parameters and examples for each executor.

## Open vs Closed Model

**Closed Model** (VU-based executors): Next iteration starts when previous finishes. System slowdown reduces throughput.
- Executors: `shared-iterations`, `per-vu-iterations`, `constant-vus`, `ramping-vus`
- Use for: Simple tests, when iteration count matters more than rate

**Open Model** (arrival-rate executors): Iterations start at fixed rate regardless of response time. Maintains throughput even under stress.
- Executors: `constant-arrival-rate`, `ramping-arrival-rate`
- Use for: Realistic RPS testing, stress testing, avoiding coordinated omission

**Recommendation:** Prefer open model (arrival-rate) for production-like load testing. It maintains target throughput even when the system slows down, giving more accurate results.

## Scenario Configuration

### Basic Scenario

```javascript
export const options = {
  scenarios: {
    my_scenario: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
    },
  },
};
```

### Common Scenario Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `executor` | string | required | Executor type |
| `startTime` | string | `"0s"` | Delay before scenario starts |
| `gracefulStop` | string | `"30s"` | Wait time for in-flight iterations |
| `exec` | string | `"default"` | Exported function to execute |
| `env` | object | `{}` | Scenario-specific environment variables |
| `tags` | object | `{}` | Scenario-specific tags |

### Multi-Scenario Orchestration

Run different load patterns simultaneously or sequentially:

```javascript
export const options = {
  scenarios: {
    // Scenario 1: Browse products (majority of traffic)
    browse: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 50,
      exec: 'browseProducts',
      tags: { scenario: 'browse' },
    },
    // Scenario 2: Purchase flow (smaller portion)
    purchase: {
      executor: 'constant-arrival-rate',
      rate: 10,
      timeUnit: '1s',
      duration: '10m',
      preAllocatedVUs: 20,
      startTime: '2m',  // Start 2 minutes after browse
      exec: 'purchaseItem',
      tags: { scenario: 'purchase' },
    },
    // Scenario 3: Spike test (starts at 5 minutes)
    spike: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '30s', target: 500 },
        { duration: '1m', target: 500 },
        { duration: '30s', target: 0 },
      ],
      startTime: '5m',
      exec: 'browseProducts',
      tags: { scenario: 'spike' },
    },
  },
};

export function browseProducts() { /* ... */ }
export function purchaseItem() { /* ... */ }
```

## Threshold Design

See [reference/thresholds.md](reference/thresholds.md) for complete threshold patterns.

### Quick Reference

```javascript
export const options = {
  thresholds: {
    // Response time
    http_req_duration: ['p(95)<500', 'p(99)<1000'],
    // Error rate
    http_req_failed: ['rate<0.01'],
    // Throughput
    http_reqs: ['rate>100'],
    // Check pass rate
    checks: ['rate>0.99'],
    // Custom metric
    my_custom_trend: ['avg<200', 'p(90)<400'],
    // Tag-based (different thresholds per endpoint)
    'http_req_duration{name:GetUser}': ['p(95)<300'],
    'http_req_duration{name:CreateOrder}': ['p(95)<800'],
  },
};
```

### Abort on Failure

```javascript
export const options = {
  thresholds: {
    http_req_duration: [{
      threshold: 'p(99)<1000',
      abortOnFail: true,
      delayAbortEval: '30s', // Wait 30s before evaluating
    }],
  },
};
```

## Custom Metrics

```javascript
import { Trend, Counter, Rate, Gauge } from 'k6/metrics';

const loginDuration = new Trend('login_duration');
const failedLogins = new Counter('failed_logins');
const successRate = new Rate('login_success_rate');
const activeUsers = new Gauge('active_users');

export const options = {
  thresholds: {
    login_duration: ['p(95)<2000'],
    login_success_rate: ['rate>0.95'],
  },
};

export default function () {
  const start = Date.now();
  const res = http.post(/* login request */);
  loginDuration.add(Date.now() - start);
  successRate.add(res.status === 200);
  if (res.status !== 200) failedLogins.add(1);
}
```

## Tags and Groups

### Tags for Metric Filtering

```javascript
// Per-request tags
http.get(url, { tags: { name: 'Homepage', type: 'page' } });

// Per-scenario tags (in options)
scenarios: {
  api_test: {
    executor: 'constant-vus',
    tags: { team: 'backend' },
  },
}
```

### Groups for Code Organization

```javascript
import { group } from 'k6';

export default function () {
  group('Authentication', () => {
    http.post('/login', credentials);
  });

  group('Browse Products', () => {
    http.get('/products');
    http.get('/products/1');
  });

  group('Checkout', () => {
    http.post('/cart', item);
    http.post('/checkout', payment);
  });
}
```

## Related Skills

- For generating test scripts: `/k6:generating-api-load-tests`
- For interpreting test results: `/k6:analyzing-test-results`
- For browser-based testing scenarios: `/k6:generating-browser-tests`
