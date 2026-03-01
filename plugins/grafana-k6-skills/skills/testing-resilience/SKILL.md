---
name: testing-resilience
description: Use when creating chaos engineering or resilience tests with k6 and xk6-disruptor. Use when the user wants to inject faults (pod failures, network delays, HTTP errors) into Kubernetes services during load tests, validate circuit breakers, or test system recovery behavior.
---

# Testing Resilience with k6

Create chaos and resilience tests using xk6-disruptor to inject faults into Kubernetes services during k6 load tests. Validates system behavior under failure conditions.

## Overview

xk6-disruptor injects faults into Kubernetes pods and services while k6 generates load, allowing you to verify:
- System survives pod failures
- Graceful degradation under network issues
- Circuit breaker behavior
- Recovery time after fault removal
- Auto-scaling response to failures

## Quick Start

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
      duration: '2m',
      preAllocatedVUs: 50,
    },
  },
  thresholds: {
    http_req_failed: ['rate<0.3'],        // Allow up to 30% errors during chaos
    http_req_duration: ['p(95)<3000'],     // Relaxed threshold during chaos
  },
};

export function setup() {
  // Inject fault: Add 500ms delay to 50% of requests
  const disruptor = new ServiceDisruptor('my-service', 'default');
  const fault = {
    averageDelay: '500ms',
    errorRate: 0.1,           // 10% of requests return errors
    errorCode: 503,
  };
  disruptor.injectHTTPFaults(fault, '60s');
}

export default function () {
  const res = http.get('http://my-service.default.svc.cluster.local:8080/api');
  check(res, {
    'not server error': (r) => r.status < 500 || r.status === 503,
  });
  sleep(0.1); // Intentional: adds backpressure per VU to avoid overwhelming the disrupted service
}
```

## Prerequisites

- **Kubernetes cluster** with target services deployed
- **k6 with xk6-disruptor** extension (or use automatic resolution)
- **RBAC permissions** for the k6 pod to manage target pods

## Disruptor Types

### ServiceDisruptor

Targets all pods behind a Kubernetes Service:

```javascript
import { ServiceDisruptor } from 'k6/x/disruptor';

const disruptor = new ServiceDisruptor('service-name', 'namespace');
```

### PodDisruptor

Targets specific pods by label selector:

```javascript
import { PodDisruptor } from 'k6/x/disruptor';

const disruptor = new PodDisruptor({
  namespace: 'default',
  select: { labels: { app: 'my-app' } },
});
```

## Fault Types

### HTTP Faults

Inject delays and errors into HTTP responses:

```javascript
disruptor.injectHTTPFaults(
  {
    averageDelay: '500ms',      // Add delay to responses
    delayVariation: '100ms',    // Delay jitter
    errorRate: 0.1,             // 10% error rate
    errorCode: 503,             // Error HTTP status code
    port: 8080,                 // Target port
    exclude: '/health',         // Exclude health check endpoint
  },
  '60s'                         // Fault duration
);
```

### gRPC Faults

Inject faults into gRPC services:

```javascript
disruptor.injectGrpcFaults(
  {
    averageDelay: '500ms',
    errorRate: 0.2,
    statusCode: 14,             // UNAVAILABLE
    statusMessage: 'service overloaded',
  },
  '60s'
);
```

### Pod Termination

Kill pods to test recovery:

```javascript
disruptor.terminatePods({
  count: 1,                     // Number of pods to terminate
  interval: '30s',              // Wait between terminations
});
```

See [reference/fault-injection.md](reference/fault-injection.md) for detailed fault configuration.

## Resilience Test Patterns

### Pattern 1: Graceful Degradation

Verify system degrades gracefully under partial failure:

```javascript
export function setup() {
  const disruptor = new ServiceDisruptor('backend-service', 'default');
  disruptor.injectHTTPFaults({
    averageDelay: '1000ms',
    errorRate: 0.2,
    errorCode: 503,
  }, '90s');
}
```

**Success criteria:** System returns degraded responses instead of cascading failures.

### Pattern 2: Circuit Breaker Validation

Verify circuit breaker activates under error conditions:

```javascript
export function setup() {
  const disruptor = new ServiceDisruptor('downstream-service', 'default');
  disruptor.injectHTTPFaults({
    errorRate: 0.8,             // 80% errors to trigger circuit breaker
    errorCode: 500,
  }, '60s');
}
```

**Success criteria:** Upstream service returns fallback responses quickly (circuit open).

### Pattern 3: Recovery Test

Verify system recovers after fault is removed:

```javascript
export const options = {
  scenarios: {
    load: {
      executor: 'constant-arrival-rate',
      rate: 50,
      timeUnit: '1s',
      duration: '5m',           // Total test: 5 minutes
      preAllocatedVUs: 30,
    },
  },
};

export function setup() {
  const disruptor = new ServiceDisruptor('my-service', 'default');
  // Inject fault for only 2 minutes (test runs 5 minutes)
  disruptor.injectHTTPFaults({
    averageDelay: '2000ms',
    errorRate: 0.5,
    errorCode: 503,
  }, '120s');  // Fault active for 2m, then 3m of recovery
}
```

**Success criteria:** Metrics return to baseline within acceptable time after fault removal.

See [reference/resilience-patterns.md](reference/resilience-patterns.md) for more patterns.

## Thresholds for Resilience Tests

Resilience tests need relaxed thresholds compared to normal load tests. Use tag-based thresholds to set different criteria for chaos and recovery phases:

```javascript
export const options = {
  thresholds: {
    // During chaos: accept higher error rates
    http_req_failed: ['rate<0.5'],                           // Up to 50% errors acceptable
    // Overall response time upper bound (includes chaos phase)
    http_req_duration: ['p(95)<5000'],                       // 5s max even during chaos
    // Strict threshold for recovery phase only (tagged requests)
    'http_req_duration{phase:recovery}': ['p(95)<500'],      // After recovery, back to normal
  },
};
```

## Related Skills

- For generating load test scripts: `/k6:generating-api-load-tests`
- For CI/CD and Kubernetes setup: `/k6:operating-k6-in-ci-cd`
- For scenario design: `/k6:designing-test-scenarios`
