---
name: designing-test-scenarios
description: Use when designing k6 load profiles, choosing executors, configuring VU patterns, setting thresholds, or planning test scenarios. Use when the user asks about stress test, spike test, soak test, breakpoint test, load model, executor selection, or pass/fail criteria.
---

# Designing k6 Test Scenarios

Design load profiles, select executors, configure thresholds, and orchestrate multi-scenario tests. This skill helps choose the right approach for your performance testing goals.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before designing, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v2.3.0 (...)`). For Docker, CI, or k6-operator, read the pinned image tag, the `setup-k6-action` `k6-version` input, or `spec.runner.image` instead.
2. If no version is visible, ask the user. Otherwise assume v2.x and record the assumption in a header comment of the generated script, for example `// Target: k6 v2.3.0`.
3. Apply the notes below when the target is v1.x.

**Version notes for this skill:**
- `--scenario` (run selected scenarios) and `--once` (single VU, single iteration) need v2.3.0+. `v1.x:` use an `__ENV.SCENARIO` switch in `options.scenarios` and `--vus 1 --iterations 1`.
- `--vus N` on its own overrides script scenarios with a `shared-iterations` N×N scenario since v2.1.0. `v1.x:` it is silently ignored when `scenarios` is defined.
- The `externally-controlled` executor was removed in v2.0.0. Never generate it.
- `K6_TAGS` and the `merge-run-tags` feature flag need v2.2.0+. Without the flag, test-wide tags from a higher-priority source replace lower ones instead of merging.
- `group()` accepts `async` callbacks only with the `async-metric-context` feature flag (v2.3.0+). Keep group callbacks synchronous by default.
- Threshold percentiles outside `0`–`100` fail at parse time since v2.1.0.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

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

### Run Selected Scenarios (`--scenario`, v2.3.0+)

Run part of a multi-scenario script without editing it. Selected scenarios keep their executor, load, `exec`, `env`, `tags`, and browser options.

```bash
k6 run --scenario browse script.js            # one scenario
k6 run --scenario browse,purchase script.js   # several
k6 run --scenario browse,purchase --once script.js  # each selected scenario once (1 VU, 1 iteration)
```

Rules:
- Names must exist in `options.scenarios` after init. `--scenario default` needs an explicit `scenarios.default`.
- Cannot be combined with `--vus`, `--duration`, `--iterations`, or `--stage`; k6 ignores those and warns.
- Thresholds tagged with an excluded scenario (`'http_req_duration{scenario:purchase}'`) are skipped with a warning. Global thresholds stay active, so a global `count` threshold sized for the full run can still fail a partial run.
- Works with `k6 cloud run` (including `--local-execution`) and `k6 archive`; the archive stores the selection.

`v1.x:` emulate selection in the script:

```javascript
const scenarios = { browse: { /* ... */ }, purchase: { /* ... */ } };
const selected = __ENV.SCENARIO ? { [__ENV.SCENARIO]: scenarios[__ENV.SCENARIO] } : scenarios;
export const options = { scenarios: selected };
// SCENARIO=browse k6 run script.js
```

### Run a Script Once (`--once`, v2.3.0+)

`k6 run --once script.js` runs one VU for one iteration while keeping a single scenario's function, environment, tags, and browser options (the scenario becomes `shared-iterations` with `startTime: '0s'`, `maxDuration: '10m'`, `gracefulStop: '30s'`). Use it for smoke checks and script validation. Bare `--once` rejects scripts with several scenarios; combine with `--scenario` to pick which ones run. It cannot be combined with `--vus`, `--duration`, `--iterations`, `--stage`, or execution segments.

`v1.x:` `k6 run --vus 1 --iterations 1 script.js` replaces every scenario with a default one, so `exec`, `env`, `tags`, and browser options are lost.

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

Test-wide tags can come from four places: `options.tags` in the script, a JSON config file, `K6_TAGS=env:staging,team:backend` (v2.2.0+), or `--tag env=staging`. By default the highest-priority source (CLI > env > script > config) **replaces** the whole map; enable the `merge-run-tags` feature flag (`k6 run --features merge-run-tags`, v2.2.0+) to merge per key instead. k6 adds the `scenario` tag automatically, so do not set it by hand.

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

Keep group callbacks synchronous. k6 rejects an `async` callback unless the experimental `async-metric-context` feature flag is enabled (`k6 run --features async-metric-context`, v2.3.0+), which also keeps the `group` tag on metrics emitted after an `await` and measures `group_duration` until the returned promise settles. `v1.x:` not available.

## Related Skills

- For generating test scripts: `/k6:generating-api-load-tests`
- For interpreting test results: `/k6:analyzing-test-results`
- For browser-based testing scenarios: `/k6:generating-browser-tests`
- For the k6 version matrix and running selected scenarios in CI: `/k6:operating-k6-in-ci-cd`
