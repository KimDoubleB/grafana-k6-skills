# k6 Operator (Kubernetes Distributed Testing) Reference

## Overview

The k6-operator is a Kubernetes operator that enables running distributed k6 load tests across multiple pods. It uses a Custom Resource Definition (CRD) called `TestRun` (previously `K6`).

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
  arguments: --out json=/tmp/results.json
  runner:
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
    image: custom-k6-image:latest   # Custom k6 image with extensions
    imagePullPolicy: Always
```

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
```

### With Istio Service Mesh

When testing services behind Istio:

```yaml
spec:
  runner:
    metadata:
      annotations:
        sidecar.istio.io/inject: "true"    # Enable Istio sidecar
```

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
