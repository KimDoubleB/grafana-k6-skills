# k6 Executors Reference

## Overview

k6 provides 6 executor types organized into two models:

**Closed Model** (VU-based): Iteration rate depends on system response time
- `shared-iterations`, `per-vu-iterations`, `constant-vus`, `ramping-vus`

**Open Model** (arrival-rate): Iteration rate is fixed regardless of response time
- `constant-arrival-rate`, `ramping-arrival-rate`

---

## shared-iterations

A fixed total number of iterations shared between VUs. Test ends when all iterations complete.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `vus` | integer | 1 | Number of VUs to run concurrently |
| `iterations` | integer | 1 | Total iterations across all VUs |
| `maxDuration` | string | `"10m"` | Max scenario duration |

**Distribution is NOT guaranteed even.** Faster VUs execute more iterations.

```javascript
export const options = {
  scenarios: {
    quick_test: {
      executor: 'shared-iterations',
      vus: 10,
      iterations: 200,
      maxDuration: '30s',
    },
  },
};
```

**Use when:** You need a fixed total iteration count and don't care about even distribution.

---

## per-vu-iterations

Each VU executes an exact number of iterations. Total = VUs x iterations.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `vus` | integer | 1 | Number of VUs to run concurrently |
| `iterations` | integer | 1 | Iterations per VU |
| `maxDuration` | string | `"10m"` | Max scenario duration |

```javascript
export const options = {
  scenarios: {
    even_distribution: {
      executor: 'per-vu-iterations',
      vus: 10,
      iterations: 20,  // Each VU does exactly 20 iterations (200 total)
      maxDuration: '1m',
    },
  },
};
```

**Use when:** Each VU needs a guaranteed number of iterations (e.g., partitioned test data).

---

## constant-vus

Fixed VU count runs for a set duration. VUs execute as many iterations as possible.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `duration` | string | required | Scenario duration |
| `vus` | integer | 1 | Number of concurrent VUs |

```javascript
export const options = {
  scenarios: {
    constant_load: {
      executor: 'constant-vus',
      vus: 50,
      duration: '5m',
    },
  },
};
```

**Use when:** Simple constant load for a fixed duration.

---

## ramping-vus

VU count changes over time via stages. VUs execute as many iterations as possible.

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `stages` | array | required | Array of `{ duration, target }` objects |
| `startVUs` | integer | 1 | Initial VU count |
| `gracefulRampDown` | string | `"30s"` | Wait time during ramp-down |

```javascript
export const options = {
  scenarios: {
    ramping_load: {
      executor: 'ramping-vus',
      startVUs: 0,
      stages: [
        { duration: '5m', target: 100 },   // Ramp up to 100
        { duration: '30m', target: 100 },  // Stay at 100
        { duration: '5m', target: 0 },     // Ramp down to 0
      ],
      gracefulRampDown: '30s',
    },
  },
};
```

**Stage behavior:** VU count transitions linearly from previous target to current target over the stage duration.

**Use when:** Load tests with ramp-up/ramp-down phases (load, stress, soak, spike tests).

---

## constant-arrival-rate

Fixed iteration rate independent of system response time (open model).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `duration` | string | required | Scenario duration |
| `rate` | integer | required | Iterations per `timeUnit` |
| `timeUnit` | string | `"1s"` | Time period for rate |
| `preAllocatedVUs` | integer | required | VUs pre-allocated before start |
| `maxVUs` | integer | preAllocatedVUs | Maximum VU count |

**Important:** Iterations start at evenly spaced intervals. At `rate: 10, timeUnit: '1s'`, one iteration starts every 100ms.

**Do NOT use `sleep()` at end of iteration** — pacing is handled by rate/timeUnit.

```javascript
export const options = {
  scenarios: {
    constant_rps: {
      executor: 'constant-arrival-rate',
      rate: 50,                // 50 iterations per second
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 20,     // Start with 20 VUs
      maxVUs: 100,             // Scale up to 100 if needed
    },
  },
};
```

**Use when:** You need a guaranteed request rate (RPS), regardless of system performance. Preferred for realistic load testing.

**Tuning preAllocatedVUs:** If too low, k6 spends time allocating VUs and may not hit target rate initially. Set to expected concurrent VU count based on `rate × average_iteration_duration`.

---

## ramping-arrival-rate

Variable iteration rate over time via stages (open model).

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `stages` | array | required | Array of `{ duration, target }` objects |
| `startRate` | integer | 0 | Initial iteration rate per `timeUnit` |
| `timeUnit` | string | `"1s"` | Time period for rate (constant for entire scenario) |
| `preAllocatedVUs` | integer | required | VUs pre-allocated before start |
| `maxVUs` | integer | preAllocatedVUs | Maximum VU count |

**Do NOT use `sleep()` at end of iteration.**

```javascript
export const options = {
  scenarios: {
    ramping_rps: {
      executor: 'ramping-arrival-rate',
      startRate: 10,
      timeUnit: '1s',
      preAllocatedVUs: 50,
      maxVUs: 200,
      stages: [
        { duration: '2m', target: 10 },   // Stay at 10 RPS
        { duration: '5m', target: 100 },  // Ramp to 100 RPS
        { duration: '5m', target: 100 },  // Stay at 100 RPS
        { duration: '2m', target: 0 },    // Ramp down to 0
      ],
    },
  },
};
```

**Use when:** Ramp request rate up/down over time. Ideal for breakpoint tests (finding system limits) and realistic traffic simulation.

---

## Executor Selection Quick Reference

| Scenario | Recommended Executor |
|----------|---------------------|
| Quick smoke test | `shared-iterations` (low vus, few iterations) |
| Each VU tests different data | `per-vu-iterations` |
| Simple constant load | `constant-vus` |
| Load/stress/soak with ramp | `ramping-vus` |
| Maintain exact RPS | `constant-arrival-rate` |
| Ramp RPS up/down | `ramping-arrival-rate` |
| Breakpoint (find limits) | `ramping-arrival-rate` |
| Spike test | `ramping-vus` (fast ramp) or `ramping-arrival-rate` |
