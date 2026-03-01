# k6 Test Types Reference

## Summary

| Test Type | VUs | Duration | Purpose |
|-----------|-----|----------|---------|
| Smoke | 2-5 | 30s-3m | Verify script works, baseline metrics |
| Load | Average production | 5-60min | Typical load performance |
| Stress | 50-100%+ above avg | 10-60min | Heavy load behavior |
| Soak | Average production | 3-72 hours | Long-running reliability |
| Spike | Very high, sudden | 2-5min | Sudden traffic burst |
| Breakpoint | Incremental | Until failure | Find system limits |

---

## Smoke Test

Minimal load to verify the script works and establish baseline metrics.

```javascript
export const options = {
  vus: 3,
  duration: '1m',
  thresholds: {
    http_req_failed: ['rate<0.01'],
    http_req_duration: ['p(95)<500'],
  },
};
```

**When to run:** After every script change. Before any other test type.

---

## Load Test (Average Load)

Assess performance under typical production load.

**Stage pattern:** Ramp-up (5-15% of total) → Plateau (5x ramp-up) → Ramp-down

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up
    { duration: '30m', target: 100 },  // Plateau
    { duration: '5m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};
```

**When to run:** Regularly after code changes. Foundation for all other test types.

**What to look for:**
- Compare response times during ramp-up vs full load
- Response time stability during plateau
- Error rate trends

---

## Stress Test

Assess performance under heavier-than-normal load (50-100%+ above average).

```javascript
export const options = {
  stages: [
    { duration: '10m', target: 200 },  // Ramp up to 2x normal
    { duration: '30m', target: 200 },  // Sustained heavy load
    { duration: '5m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<1000'],
    http_req_failed: ['rate<0.05'],
  },
};
```

**When to run:** After successful load tests. Before anticipated high-traffic events.

**What to look for:**
- Performance degradation rate
- Error rate increase patterns
- Recovery behavior during ramp-down

---

## Soak Test

Test reliability and performance over extended periods. Same load as load test but much longer duration.

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Ramp up
    { duration: '8h', target: 100 },   // Extended plateau (hours!)
    { duration: '5m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],
    http_req_failed: ['rate<0.01'],
  },
};
```

**When to run:** After successful load and stress tests. Before production deployments.

**What to look for:**
- Memory leaks (gradual increase in response times)
- Database connection pool exhaustion
- Log/storage growth issues
- Gradual metric degradation over time

---

## Spike Test

Verify system survives sudden, massive traffic bursts.

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 2000 },  // Fast ramp to extreme load
    { duration: '1m', target: 0 },     // Quick ramp down
  ],
};
```

**Alternative — multiple spikes:**

```javascript
export const options = {
  stages: [
    { duration: '1m', target: 100 },    // Normal load
    { duration: '30s', target: 1500 },  // Spike 1
    { duration: '30s', target: 100 },   // Recovery
    { duration: '30s', target: 1500 },  // Spike 2
    { duration: '1m', target: 0 },      // Ramp down
  ],
};
```

**When to run:** For systems expecting sudden traffic (sales events, broadcasts, deadlines).

**What to look for:**
- System survival (does it crash?)
- Auto-scaling response time
- Recovery time after spike
- Error patterns during peak

---

## Breakpoint Test

Find system capacity limits by continuously increasing load until failure.

**Recommended executor:** `ramping-arrival-rate` (maintains throughput despite degradation)

```javascript
export const options = {
  scenarios: {
    breakpoint: {
      executor: 'ramping-arrival-rate',
      preAllocatedVUs: 100,
      maxVUs: 2000,
      stages: [
        { duration: '2h', target: 20000 },  // Slow continuous ramp
      ],
    },
  },
  thresholds: {
    http_req_failed: [{
      threshold: 'rate>0.1',
      abortOnFail: true,
      delayAbortEval: '1m',
    }],
  },
};
```

**When to run:** To discover system capacity limits. After tuning system to verify new limits.

**What to look for (progressive failure stages):**
1. Degraded performance → response times increase
2. Troublesome → severe user experience degradation
3. Timeouts → requests start timing out
4. Errors → HTTP error codes appear
5. System failure → complete collapse

**Warning:** Avoid in elastic cloud environments without disabling auto-scaling, or you may only discover your cloud bill limit.

---

## Test Type Decision Guide

```
What do you want to learn?
│
├─ "Does my script work?" → Smoke Test
│
├─ "How does it perform normally?" → Load Test
│
├─ "Can it handle peak traffic?" → Stress Test
│
├─ "Is it reliable over time?" → Soak Test
│
├─ "Can it survive sudden bursts?" → Spike Test
│
└─ "Where does it break?" → Breakpoint Test
```

**Recommended progression:** Smoke → Load → Stress → Soak → Spike → Breakpoint
