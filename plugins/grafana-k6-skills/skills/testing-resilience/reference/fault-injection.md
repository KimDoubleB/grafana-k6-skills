# Fault Injection Reference

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

### Running in Cluster

For xk6-disruptor to work, k6 must run **inside the Kubernetes cluster** (not locally). Options:
- k6-operator TestRun
- Pod with k6 + xk6-disruptor image
- CI/CD runner with cluster access

## Fault Duration vs Test Duration

```
Test Duration:     |========================== 5 min ============================|
Fault Duration:    |======== 2 min ========|
                   ↑ Fault injected         ↑ Fault removed    ↑ Test ends

Phases:            |--- Chaos Phase ------|--- Recovery Phase ---|
```

Set fault duration shorter than test duration to observe both degradation and recovery.
