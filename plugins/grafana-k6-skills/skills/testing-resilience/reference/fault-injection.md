# Fault Injection Reference

> xk6-disruptor is archived (February 2026) and only works with k6 **v1.x** binaries. API below is for v0.3.17, the final release.

## ServiceDisruptor

Targets all pods behind a Kubernetes Service.

```javascript
import { ServiceDisruptor } from 'k6/x/disruptor';

const disruptor = new ServiceDisruptor('service-name', 'namespace');
```

### injectHTTPFaults(fault, duration)

| Parameter | Type | Description |
|-----------|------|-------------|
| `averageDelay` | string | Average delay added to responses (e.g., `'500ms'`) |
| `delayVariation` | string | Variation around averageDelay (e.g., `'100ms'`) |
| `errorRate` | float | Fraction of requests returning errors (0.0-1.0) |
| `errorCode` | integer | HTTP status code for injected errors |
| `errorBody` | string | Response body for injected errors |
| `port` | integer | Target port (required if service has multiple ports) |
| `exclude` | string | URL path prefix to exclude from faults |

```javascript
disruptor.injectHTTPFaults({
  averageDelay: '500ms',
  delayVariation: '100ms',
  errorRate: 0.1,
  errorCode: 503,
  errorBody: '{"error": "service unavailable"}',
  port: 8080,
  exclude: '/health',
}, '60s');
```

### injectGrpcFaults(fault, duration)

| Parameter | Type | Description |
|-----------|------|-------------|
| `averageDelay` | string | Average delay added to responses |
| `delayVariation` | string | Variation around averageDelay |
| `errorRate` | float | Fraction of requests returning errors (0.0-1.0) |
| `statusCode` | integer | gRPC status code for errors |
| `statusMessage` | string | gRPC status message for errors |
| `port` | integer | Target port |
| `exclude` | string | Service method to exclude |

```javascript
disruptor.injectGrpcFaults({
  averageDelay: '500ms',
  errorRate: 0.2,
  statusCode: 14,       // UNAVAILABLE
  statusMessage: 'injected fault',
}, '60s');
```

### terminatePods(options)

| Parameter | Type | Description |
|-----------|------|-------------|
| `count` | integer | Number of pods to terminate |
| `interval` | string | Wait time between terminations |

```javascript
disruptor.terminatePods({
  count: 1,
  interval: '30s',
});
```

## PodDisruptor

Targets pods by label selector.

```javascript
import { PodDisruptor } from 'k6/x/disruptor';

const disruptor = new PodDisruptor({
  namespace: 'default',
  select: {
    labels: { app: 'my-app', version: 'v1' },
  },
});
```

Same methods as ServiceDisruptor: `injectHTTPFaults()`, `injectGrpcFaults()`, `terminatePods()`.

Additional method:

### targets()

Returns list of target pod names:

```javascript
const pods = disruptor.targets();
console.log(`Targeting ${pods.length} pods:`, pods);
```

## Kubernetes Requirements

### RBAC

The k6 pod needs permissions to manage target pods:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k6-disruptor
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/ephemeralcontainers"]
    verbs: ["get", "list", "patch", "update", "delete"]
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list"]
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: k6-disruptor-binding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: k6-disruptor
subjects:
  - kind: ServiceAccount
    name: default
    namespace: k6-test
```

### Where k6 Runs

xk6-disruptor talks to the Kubernetes API (it injects an agent as an ephemeral container into the target pods) and the load part of the script must reach the service. Both are satisfied by:
- A k6-operator `TestRun` with `runner.image: grafana/k6:v1` and `K6_ENABLE_COMMUNITY_EXTENSIONS=true` in `runner.env` (or a custom v1 image with the extension baked in)
- A pod or CI runner inside the cluster using a service account with the RBAC above
- A workstation with a kubeconfig, when the service is exposed (the official guide runs the examples this way)

### Binary Requirements

| Way to run | Command | Notes |
|------------|---------|-------|
| k6 v1.8.x auto-resolution | `K6_ENABLE_COMMUNITY_EXTENSIONS=true k6 run chaos.js` | Community tier; the variable is mandatory on v1.x and does not exist on v2.x |
| xk6 build | `xk6 build v1.8.2 --with github.com/grafana/xk6-disruptor` | Fails for `xk6 build v2.x` (module path mismatch) |
| Prebuilt binary | `xk6-disruptor run chaos.js` from the v0.3.17 GitHub release | Bundles its own k6 v1.x |
| k6 v2.x | not possible | Use a `grafana/k6:v1` image for the chaos job or a cluster-native chaos tool |

## Fault Duration vs Test Duration

```
Test Duration:     |========================== 5 min ============================|
Fault Duration:    |======== 2 min ========|
                   ↑ Fault injected         ↑ Fault removed    ↑ Test ends

Phases:            |--- Chaos Phase ------|--- Recovery Phase ---|
```

Set fault duration shorter than test duration to observe both degradation and recovery.
