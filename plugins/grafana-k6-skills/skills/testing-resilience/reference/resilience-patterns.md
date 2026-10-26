# Resilience Test Patterns Reference

> All patterns require a k6 **v1.x** binary with xk6-disruptor v0.3.17 (archived). On k6 v1.8.x run them with `K6_ENABLE_COMMUNITY_EXTENSIONS=true`. On k6 v2.x the extension cannot be loaded; see SKILL.md for alternatives.

## Pattern 1: Graceful Degradation

Verify the system degrades gracefully under partial failure instead of cascading.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { ServiceDisruptor } from 'k6/x/disruptor';

export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '3m',
      preAllocatedVUs: 50,
      maxVUs: 100,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.5'],            // Allow up to 50% errors
    http_req_duration: ['p(95)<5000'],         // 5s max during degradation
    'http_req_duration{phase:recovery}': ['p(95)<500'],  // Back to normal after
  },
};

export function setup() {
  const disruptor = new ServiceDisruptor('backend-api', 'default');
  disruptor.injectHTTPFaults({
    averageDelay: '1000ms',
    delayVariation: '200ms',
    errorRate: 0.3,
    errorCode: 503,
    exclude: '/health',
  }, '90s');  // Fault for 90s, test runs 3m → 90s recovery
}

export default function () {
  const res = http.get('http://backend-api.default.svc.cluster.local:8080/api/data');
  check(res, {
    'returns valid response': (r) => r.status === 200 || r.status === 503,
    'no cascading failure': (r) => r.status !== 502,
    'responds within timeout': (r) => r.timings.duration < 10000,
  });
  sleep(0.1);
}
```

**Success criteria:**
- System returns 503 (expected) not 502 (cascade)
- Response times degrade but stay within upper bound
- No complete service outage
- Metrics return to baseline during recovery phase

---

## Pattern 2: Circuit Breaker Validation

Verify circuit breaker activates under sustained errors and returns fallback responses.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';
import { ServiceDisruptor } from 'k6/x/disruptor';

const circuitOpenResponses = new Counter('circuit_open_responses');
const responseTime = new Trend('response_time_trend');

export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '4m',
      preAllocatedVUs: 30,
      maxVUs: 60,
    },
  },
  thresholds: {
    circuit_open_responses: ['count>0'],       // Circuit breaker MUST activate
    http_req_duration: ['p(95)<3000'],         // Fast fallback expected
  },
};

export function setup() {
  const disruptor = new ServiceDisruptor('downstream-service', 'default');
  // High error rate to trigger circuit breaker
  disruptor.injectHTTPFaults({
    errorRate: 0.8,
    errorCode: 500,
    averageDelay: '3000ms',
  }, '120s');  // 2 min of faults, then 2 min recovery
}

export default function () {
  const res = http.get('http://api-gateway.default.svc.cluster.local:8080/api/data');
  responseTime.add(res.timings.duration);

  // Detect circuit breaker activation via fallback response
  const isFallback = res.status === 200 && res.json() &&
    (res.json().source === 'cache' || res.json().fallback === true);

  if (isFallback) {
    circuitOpenResponses.add(1);
  }

  check(res, {
    'not a timeout': (r) => r.timings.duration < 10000,
    'valid response or fallback': (r) => r.status === 200 || r.status === 503,
  });
  sleep(0.1);
}
```

**Success criteria:**
- `circuit_open_responses` count > 0 (circuit breaker activated)
- Response times drop after circuit opens (fast fallback)
- After fault removal, circuit closes and normal responses resume

**Phases observed:**
```
|-- Closed --|-- Open (fallback) --|-- Half-Open --|-- Closed --|
|  Errors    |  Fast fallback      |  Trial reqs   |  Normal    |
```

---

## Pattern 3: Recovery Time Measurement

Measure how quickly the system recovers after faults are removed.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Trend, Rate } from 'k6/metrics';
import { ServiceDisruptor } from 'k6/x/disruptor';

const recoveryLatency = new Trend('recovery_phase_latency');
const recoveryErrors = new Rate('recovery_phase_errors');

export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 30,
      maxVUs: 60,
    },
  },
  thresholds: {
    recovery_phase_latency: ['p(95)<500'],     // Must recover to baseline
    recovery_phase_errors: ['rate<0.01'],       // Near-zero errors after recovery
    http_req_duration: ['p(95)<5000'],          // Overall upper bound
  },
};

export function setup() {
  const disruptor = new ServiceDisruptor('my-service', 'default');
  // Fault for 2 min → 3 min recovery observation
  disruptor.injectHTTPFaults({
    averageDelay: '2000ms',
    delayVariation: '500ms',
    errorRate: 0.5,
    errorCode: 503,
  }, '120s');

  return { faultEndTime: Date.now() + 120000 };
}

export default function (data) {
  const res = http.get('http://my-service.default.svc.cluster.local:8080/api');
  const isRecoveryPhase = Date.now() > data.faultEndTime;

  if (isRecoveryPhase) {
    recoveryLatency.add(res.timings.duration);
    recoveryErrors.add(res.status >= 500 ? 1 : 0);
  }

  check(res, {
    'status ok or expected error': (r) => r.status === 200 || r.status === 503,
  });
  sleep(0.1);
}
```

**Success criteria:**
- `recovery_phase_latency` p(95) returns to baseline (< 500ms)
- `recovery_phase_errors` rate drops to near zero
- Recovery happens within acceptable window (e.g., 30 seconds)

---

## Pattern 4: Pod Failure and Auto-Scaling

Test system behavior when pods are terminated and verify Kubernetes auto-scaling replaces them.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';
import { ServiceDisruptor } from 'k6/x/disruptor';

const errorRate = new Rate('error_rate');
const latency = new Trend('request_latency');

export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      maxVUs: 100,
    },
    disrupt: {
      executor: 'shared-iterations',
      iterations: 1,
      vus: 1,
      exec: 'disrupt',
      startTime: '30s',   // Start disruption after 30s baseline
    },
  },
  thresholds: {
    error_rate: ['rate<0.3'],                  // Allow transient errors
    request_latency: ['p(99)<10000'],          // Upper bound during disruption
  },
};

export function disrupt() {
  const disruptor = new ServiceDisruptor('my-service', 'default');
  // Terminate 1 pod every 30 seconds
  disruptor.terminatePods({
    count: 1,
    interval: '30s',
  });
}

export default function () {
  const res = http.get('http://my-service.default.svc.cluster.local:8080/api');
  errorRate.add(res.status >= 500 || res.status === 0 ? 1 : 0);
  latency.add(res.timings.duration);

  check(res, {
    'service available': (r) => r.status !== 0,
    'not server error': (r) => r.status < 500,
  });
  sleep(0.1);
}
```

**Success criteria:**
- Error rate stays below threshold despite pod termination
- Kubernetes replaces terminated pods (check with `kubectl get pods -w`)
- Service remains available throughout (no total outage)
- Response times stabilize after replacement pods are ready

---

## Pattern 5: Failover Testing

Test traffic routing to healthy instances when some become unavailable.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { ServiceDisruptor } from 'k6/x/disruptor';

export const options = {
  scenarios: {
    baseline: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '1m',
      preAllocatedVUs: 30,
      exec: 'loadTest',
      startTime: '0s',
    },
    failover: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '2m',
      preAllocatedVUs: 30,
      exec: 'loadTest',
      startTime: '1m',           // Start after baseline
    },
    inject: {
      executor: 'shared-iterations',
      iterations: 1,
      vus: 1,
      exec: 'injectFault',
      startTime: '1m',           // Inject at same time as failover phase
    },
  },
  thresholds: {
    'http_req_failed{scenario:baseline}': ['rate<0.01'],   // Baseline: near-zero errors
    'http_req_failed{scenario:failover}': ['rate<0.1'],    // Failover: some errors ok
    'http_req_duration{scenario:failover}': ['p(95)<2000'],
  },
};

export function injectFault() {
  const disruptor = new ServiceDisruptor('primary-service', 'default');
  // Make primary service return errors
  disruptor.injectHTTPFaults({
    errorRate: 1.0,
    errorCode: 503,
  }, '120s');
}

export function loadTest() {
  // Request goes through a load balancer or gateway that should failover
  const res = http.get('http://api-gateway.default.svc.cluster.local:8080/api/data');
  check(res, {
    'response received': (r) => r.status !== 0,
    'successful response': (r) => r.status === 200,
  });
  sleep(0.1);
}
```

**Success criteria:**
- Baseline phase has near-zero errors
- After primary failure, traffic routes to secondary/healthy instances
- Error rate during failover stays within threshold
- No sustained outage period

---

## Pattern 6: Baseline Comparison with SKIP_FAULTS

Run the same test with and without faults to measure exact degradation impact.

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { ServiceDisruptor } from 'k6/x/disruptor';

const SKIP_FAULTS = (__ENV.SKIP_FAULTS === '1');

export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '30s',
      preAllocatedVUs: 10,
      maxVUs: 100,
    },
    disrupt: {
      executor: 'shared-iterations',
      iterations: 1,
      vus: 1,
      exec: 'disrupt',
      startTime: '0s',
    },
  },
};

export function disrupt() {
  if (SKIP_FAULTS) {
    console.log('Skipping fault injection (baseline run)');
    return;
  }

  const disruptor = new ServiceDisruptor('my-app', 'default');
  disruptor.injectHTTPFaults({
    averageDelay: '50ms',
    errorCode: 500,
    errorRate: 0.1,
  }, '30s');
}

export default function () {
  const res = http.get('http://my-app.default.svc.cluster.local:8080/api');
  check(res, {
    'status is 200': (r) => r.status === 200,
  });
  sleep(0.1);
}
```

**Usage:**
```bash
# Run baseline (no faults)
K6_ENABLE_COMMUNITY_EXTENSIONS=true SKIP_FAULTS=1 k6 run test.js --out json=baseline.json

# Run with faults
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run test.js --out json=fault.json

# Compare results
# Baseline: ~100ms p95, 0% errors
# With faults: ~150ms p95, 10% errors
```

**What to compare:**
| Metric | Baseline | With Faults | Acceptable Delta |
|--------|----------|-------------|------------------|
| p(95) latency | 100ms | 150ms | < 2x baseline |
| Error rate | 0% | 10% | = injected rate |
| Throughput | 100 req/s | 90 req/s | > 70% baseline |

---

## Multi-Scenario Architecture

Use separate `exec` functions for load generation and fault injection in the same test:

```javascript
export const options = {
  scenarios: {
    // Load generation runs the entire test
    load: {
      executor: 'constant-arrival-rate',
      rate: 100,
      timeUnit: '1s',
      duration: '5m',
      preAllocatedVUs: 50,
      maxVUs: 100,
      exec: 'default',
    },
    // Fault injection runs once at a specific time
    disrupt: {
      executor: 'shared-iterations',
      iterations: 1,
      vus: 1,
      exec: 'disrupt',
      startTime: '30s',       // Start after 30s baseline
    },
  },
};
```

**Timeline:**
```
0s        30s                   2m30s              5m
|---------|---------------------|------------------|
 Baseline   Fault Active          Recovery
 (clean)    (degraded)            (back to normal)
```

This pattern is preferred over `setup()` injection because:
- Captures clean baseline metrics before faults
- Fault injection timing is precise via `startTime`
- Load and disruption scenarios run independently

---

## Threshold Strategy for Resilience Tests

### Relaxed Overall + Strict Recovery

```javascript
export const options = {
  thresholds: {
    // Overall: relaxed (includes chaos phase)
    http_req_failed: ['rate<0.5'],
    http_req_duration: ['p(95)<5000'],

    // Tag-based: strict for recovery phase
    'http_req_duration{phase:recovery}': ['p(95)<500'],
    'http_req_failed{phase:recovery}': ['rate<0.01'],
  },
};
```

### Abort on Catastrophic Failure

```javascript
export const options = {
  thresholds: {
    // Abort if system completely fails
    http_req_failed: [{
      threshold: 'rate<0.95',
      abortOnFail: true,
      delayAbortEval: '10s',
    }],
  },
};
```

### Recommended Thresholds by Pattern

| Pattern | Error Rate | p(95) Latency | Notes |
|---------|-----------|---------------|-------|
| Graceful Degradation | < 50% | < 5s | Errors expected, no cascade |
| Circuit Breaker | < 30% | < 3s | Fast fallback expected |
| Recovery | < 1% (recovery) | < 500ms (recovery) | Strict after fault removal |
| Pod Failure | < 30% | < 10s | Transient errors during reschedule |
| Failover | < 10% | < 2s | Quick reroute expected |
