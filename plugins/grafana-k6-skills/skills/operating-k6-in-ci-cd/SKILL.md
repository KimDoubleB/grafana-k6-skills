---
name: operating-k6-in-ci-cd
description: Use when setting up k6 in CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins), configuring distributed testing with k6-operator on Kubernetes, or managing k6 extensions (xk6). Use when the user wants to automate performance tests in a pipeline or run distributed load tests.
---

# Operating k6 in CI/CD

Configure k6 for CI/CD pipelines, distributed testing on Kubernetes, and extension management.

## Choose Your Path

- **CI/CD Pipeline Setup** → See [reference/ci-cd-pipelines.md](reference/ci-cd-pipelines.md)
  - GitHub Actions, GitLab CI, Jenkins
  - Docker-based execution
  - Threshold-based pass/fail in pipelines

- **Distributed Testing (Kubernetes)** → See [reference/k6-operator.md](reference/k6-operator.md)
  - k6-operator installation
  - TestRun CRD configuration
  - Scaling load across pods

- **Extensions (xk6)** → See [reference/extensions.md](reference/extensions.md)
  - Building custom k6 binaries
  - Official and community extensions
  - Automatic extension resolution

## Quick Start: GitHub Actions

```yaml
name: k6 Load Test
on: [push]
jobs:
  k6:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
      - uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
```

## Quick Start: Docker

```bash
docker run --rm -i grafana/k6 run - < script.js

# With local script file
docker run --rm -v $(pwd):/scripts grafana/k6 run /scripts/test.js

# With environment variables
docker run --rm -v $(pwd):/scripts \
  -e BASE_URL=https://staging.example.com \
  grafana/k6 run /scripts/test.js
```

## Key Concepts

### Threshold-Based Pass/Fail

k6 exits with code 99 if thresholds fail. CI/CD pipelines use this to pass/fail the build:

```javascript
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500'],  // Fails pipeline if p95 > 500ms
    http_req_failed: ['rate<0.01'],    // Fails pipeline if error rate > 1%
  },
};
```

### Environment-Specific Configuration

```javascript
const BASE_URL = __ENV.BASE_URL || 'https://api.example.com';
const TARGET_VUS = parseInt(__ENV.TARGET_VUS) || 10;

export const options = {
  vus: TARGET_VUS,
  duration: __ENV.DURATION || '5m',
};
```

### Grafana Cloud k6 Integration

```bash
# Run locally, send results to cloud
K6_CLOUD_TOKEN=$TOKEN k6 cloud run --local-execution script.js

# Run entirely in cloud
K6_CLOUD_TOKEN=$TOKEN k6 cloud run script.js
```

## Related Skills

- For generating test scripts: `/k6:generating-api-load-tests`
- For analyzing results: `/k6:analyzing-test-results`
- For resilience testing on Kubernetes: `/k6:testing-resilience`
