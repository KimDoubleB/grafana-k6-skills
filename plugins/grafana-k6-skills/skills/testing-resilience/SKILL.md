---
name: testing-resilience
description: Use when creating chaos engineering or resilience tests with k6 and xk6-disruptor. Use when the user wants to inject faults (pod failures, network delays, HTTP errors) into Kubernetes services during load tests, validate circuit breakers, or test system recovery behavior.
---

# Testing Resilience with k6

Create chaos and resilience tests using xk6-disruptor to inject faults into Kubernetes services during k6 load tests. Validates system behavior under failure conditions.

> **xk6-disruptor is archived.** Grafana Labs archived the repository in February 2026 (last release v0.3.17, October 2025). It is still listed in the k6 extension registry (community tier), but receives no fixes and is built against the k6 **v1** Go module path, so it only works with **k6 v1.x binaries**. Tell the user this before generating a test, and confirm they can run k6 v1.8.x.

## k6 Version Compatibility

Written for **k6 v1.8.2** running xk6-disruptor v0.3.17. This is the one skill in the plugin whose primary target is the v1.x maintenance line; k6 v2.x cannot load the extension. Before generating, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v1.8.2 (...)` or `k6 v2.3.0 (...)`). For Docker, CI, or k6-operator, read the pinned image tag, the `setup-k6-action` `k6-version` input, or `spec.runner.image`.
2. If the environment runs k6 v2.x, stop and offer the options below; do not emit a `k6/x/disruptor` script that cannot provision. If no version is visible, ask.
3. Record the resolved version in a header comment, for example `// Target: k6 v1.8.2 + xk6-disruptor v0.3.17`.

**Version notes for this skill:**
- k6 v1.x needs `K6_ENABLE_COMMUNITY_EXTENSIONS=true` for the `k6/x/disruptor` import to auto-resolve (the extension is community tier). Alternatives: `grafana/k6:v1` plus the variable, `xk6 build v1.8.2 --with github.com/grafana/xk6-disruptor`, or the prebuilt `xk6-disruptor` binary from the GitHub release.
- k6 v2.x: the extension's `go.k6.io/k6` import does not match `go.k6.io/k6/v2`, so neither auto-resolution nor `xk6 build v2.x` can include it. Options: (a) run the chaos test with a pinned `grafana/k6:v1` image while the rest of the suite uses v2; (b) inject faults with a cluster-native tool (Chaos Mesh, LitmusChaos, Gremlin) and keep k6 v2 for load and thresholds, orchestrating the two from the pipeline; (c) fork xk6-disruptor and update the module path.
- The official guide's `--summary-mode=legacy` flag was removed in k6 v2.0.0 and is unnecessary on v1.x.
- Everything else in this skill (`ServiceDisruptor`, `PodDisruptor`, fault options, RBAC) is unchanged since v0.3.x.

For the full feature/version matrix see `/k6:operating-k6-in-ci-cd` (reference/version-compatibility.md).

## Overview

xk6-disruptor injects faults into Kubernetes pods and services while k6 generates load, allowing you to verify:
- System survives pod failures
- Graceful degradation under network issues
- Circuit breaker behavior
- Recovery time after fault removal
- Auto-scaling response to failures

## Quick Start

```javascript
// Target: k6 v1.8.2 + xk6-disruptor v0.3.17 (community extension; needs K6_ENABLE_COMMUNITY_EXTENSIONS=true on v1.x)
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

Run it with a k6 v1.x binary:

```bash
# Auto-resolution on k6 v1.8.x (community tier needs the opt-in variable)
K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run chaos.js

# Docker, pinned to the v1 line
docker run --rm -i -e K6_ENABLE_COMMUNITY_EXTENSIONS=true -v ~/.kube:/root/.kube grafana/k6:v1 run - < chaos.js

# Or build once with xk6
xk6 build v1.8.2 --with github.com/grafana/xk6-disruptor && ./k6 run chaos.js
```

## Prerequisites

- **Kubernetes cluster** with target services deployed, reachable from where k6 runs, and a kubeconfig the k6 process can use
- **k6 v1.x with xk6-disruptor** (auto-resolution with `K6_ENABLE_COMMUNITY_EXTENSIONS=true`, an xk6 build, or the prebuilt `xk6-disruptor` binary). k6 v2.x is not supported by the extension
- **RBAC permissions** for the k6 identity to list services/pods and patch pods (ephemeral containers)

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
- For CI/CD and Kubernetes setup, pinning `grafana/k6:v1`, and the version matrix: `/k6:operating-k6-in-ci-cd`
- For scenario design: `/k6:designing-test-scenarios`
