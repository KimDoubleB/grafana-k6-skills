# k6 Operator (Kubernetes Distributed Testing) Reference

## Overview

The k6-operator is a Kubernetes operator that enables running distributed k6 load tests across multiple pods. It uses a Custom Resource Definition (CRD) called `TestRun` (previously `K6`).

### Operator Version and Runner k6 Version

The operator release decides which k6 runs your script unless you pin `spec.runner.image`:

| k6-operator | Released | Default runner k6 | Notable |
|-------------|----------|-------------------|---------|
| v1.7.0 | 2026-10 | v2.3.0 | — |
| v1.6.0 | 2026-08 | v2.2.0 | `.spec.args` array field |
| v1.5.0 | 2026-06 | **v2.0.0** (first v2 runner) | `.spec.scuttle` deprecated, `schedulerName` |
| v1.4.0 | 2026-04 | v1.x | Default metric tags, PrivateLoadZone secrets |
| v1.3.0 | 2026-03 | v1.6.1 | Option to skip the initializer pod |

Check what you have: `kubectl get deploy -n k6-operator-system k6-operator-controller-manager -o jsonpath='{.spec.template.spec.containers[0].image}'` and `kubectl get testrun <name> -o jsonpath='{.spec.runner.image}'`. Scripts that rely on v1-only behavior (removed flags, `externally-controlled`, extensions built for `go.k6.io/k6` v1) need `runner.image: grafana/k6:v1` on operator v1.5.0+.

## Installation

### Using Helm

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install k6-operator grafana/k6-operator \
  --namespace k6-operator-system \
  --create-namespace
```

### Using kubectl

```bash
# Install CRDs and operator
kubectl apply -f https://github.com/grafana/k6-operator/releases/latest/download/bundle.yaml
```

### Verify Installation

```bash
kubectl get pods -n k6-operator-system
# Should see k6-operator-controller-manager running
```

## TestRun CRD

### Basic TestRun

```yaml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: k6-load-test
spec:
  parallelism: 4                  # Number of k6 pods
  script:
    configMap:
      name: k6-test-script        # ConfigMap containing the test script
      file: test.js               # Script filename in ConfigMap
```

### Creating the ConfigMap

```bash
# From file
kubectl create configmap k6-test-script --from-file=test.js=./tests/load-test.js

# Or inline
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: k6-test-script
data:
  test.js: |
    import http from 'k6/http';
    import { check } from 'k6';

    export const options = {
      vus: 50,
      duration: '5m',
    };

    export default function () {
      const res = http.get('http://my-service.default.svc.cluster.local:8080/api');
      check(res, { 'status 200': (r) => r.status === 200 });
    }
EOF
```

### Running the Test

```bash
kubectl apply -f testrun.yaml

# Monitor progress
kubectl get testrun k6-load-test -w

# Check pod logs
kubectl logs -l k6_cr=k6-load-test -f
```

### Cleanup

```bash
kubectl delete testrun k6-load-test
```

## Advanced Configuration

### With Arguments and Environment Variables

```yaml
apiVersion: k6.io/v1alpha1
kind: TestRun
metadata:
  name: k6-load-test
spec:
  parallelism: 4
  script:
    configMap:
      name: k6-test-script
      file: test.js
  # Operator v1.6.0+: args is an array, so values with spaces, quotes, or `$` need no escaping.
  # When args is present it overrides the older single-string `arguments` field.
  args:
    - --out
    - json=/tmp/results.json
    - --tag
    - note=values with spaces "work" here
    - -e
    - BASE_URL=$(BASE_URL)        # Kubernetes expands env references
  # Older operators: arguments: --out json=/tmp/results.json
  runner:
    image: grafana/k6:2.3.0        # pin the k6 version explicitly
    env:
      - name: BASE_URL
        value: "http://my-service.default.svc.cluster.local:8080"
      - name: API_KEY
        valueFrom:
          secretKeyRef:
            name: api-secrets
            key: api-key
    resources:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "200m"
        memory: "256Mi"
```

### With Extensions (Custom Image)

```yaml
spec:
  runner:
    image: custom-k6-image:2.3.0   # Custom k6 image with extensions, built with xk6 against go.k6.io/k6/v2
    imagePullPolicy: Always
```

With automatic extension resolution the runner can also provision official and community extensions on demand from plain `import 'k6/x/...'` lines, as long as the pods can reach the k6 build service. `v1.x:` community extensions additionally need `K6_ENABLE_COMMUNITY_EXTENSIONS=true` in `runner.env`.

### With Grafana Cloud k6

```yaml
spec:
  parallelism: 4
  script:
    configMap:
      name: k6-test-script
      file: test.js
  arguments: --out cloud
  runner:
    env:
      - name: K6_CLOUD_TOKEN
        valueFrom:
          secretKeyRef:
            name: k6-cloud-secrets
            key: token
      - name: K6_CLOUD_STACK_ID      # k6 v2 requires a stack for k6 cloud commands; harmless for --out cloud
        valueFrom:
          secretKeyRef:
            name: k6-cloud-secrets
            key: stack-id
```

The operator can also read the token from a secret labeled `k6cloud=token` in its own namespace (`kubectl -n k6-operator-system create secret generic my-cloud-token --from-literal=token=... && kubectl -n k6-operator-system label secret my-cloud-token k6cloud=token`). If a v2 runner logs an authentication error that mentions the stack, add `K6_CLOUD_STACK_ID` as above.

### With Istio Service Mesh

When testing services behind Istio:

```yaml
spec:
  runner:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"    # Enable Istio sidecar
```

`.spec.scuttle` (the old sidecar-termination helper) is deprecated since operator v1.5.0; follow the operator's Istio documentation for sidecar lifecycle instead of relying on it.

## Parallelism and Load Distribution

Each pod runs the full test script independently. If your script defines `vus: 100` and `parallelism: 4`, each pod runs 100 VUs (400 total VUs).

**To distribute load evenly:** Divide VUs by parallelism in your script or use environment variables:

```javascript
const PARALLELISM = parseInt(__ENV.K6_PARALLELISM) || 1;
const TOTAL_VUS = 400;

export const options = {
  vus: Math.ceil(TOTAL_VUS / PARALLELISM),
  duration: '5m',
};
```

Or use scenarios with `constant-arrival-rate` for more predictable distribution.

## TestRun Lifecycle

```
Created → Initialized → Started → Finished
                                  ↓
                              (cleanup)
```

**Monitor status:**
```bash
kubectl get testrun -w
# NAME           STAGE       STATUS    AGE
# k6-load-test   started    running   2m
```

## Troubleshooting

### Common Issues

| Problem | Solution |
|---------|----------|
| Pods stuck in Pending | Check resource limits, node capacity |
| ConfigMap not found | Verify ConfigMap exists in same namespace |
| Connection refused | Check target service DNS and network policies |
| Uneven load | Use arrival-rate executors for better distribution |
| OOMKilled | Increase memory limits in runner.resources |
| `unknown flag`, `--no-summary`, `externally-controlled`, or `K6_ENABLE_COMMUNITY_EXTENSIONS` errors after an operator upgrade | Runner is now k6 v2.x (operator v1.5.0+). Update the script or pin `runner.image: grafana/k6:v1` |
| Extension import fails to provision on a v2 runner | The extension is built for `go.k6.io/k6` v1 (for example the archived xk6-disruptor). Use `grafana/k6:v1` or a v2-compatible extension |
| Initializer job fails before any runner starts | Operator v1.5.0+ logs the k6 error from the initializer pod; run the same script locally with `k6 inspect` first |
