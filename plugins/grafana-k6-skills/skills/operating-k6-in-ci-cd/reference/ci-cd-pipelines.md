# k6 CI/CD Pipeline Reference

## Pin and Record the k6 Version

Every pipeline should make the k6 version explicit and print it, so results are reproducible and the version-specific notes in this plugin can be applied:

| CI system | How to pin | How to print |
|-----------|------------|--------------|
| GitHub Actions | `grafana/setup-k6-action` with `k6-version: '2.3.0'` (unset = latest stable) | `- run: k6 version` |
| Docker-based jobs (GitLab, Jenkins, others) | Image tag `grafana/k6:2.3.0`; `grafana/k6:v2` / `grafana/k6:v1` float within a major, `latest` = newest v2.x; `*-with-browser` adds Chromium | `k6 version` as the first script line |
| Manual install (apt/rpm/tarball) | Download a release asset, e.g. `k6-v2.3.0-linux-amd64.tar.gz`; the apt repository tracks the latest stable | `k6 version` |

`k6 version` prints `k6 v2.3.0 (commit/<sha>, go<version>, <os>/<arch>)`; the first token after `k6` is the version.

## GitHub Actions

### Using Official Actions

```yaml
# Target: k6 v2.3.0
name: k6 Performance Test
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  k6-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup k6
        uses: grafana/setup-k6-action@v1
        with:
          k6-version: '2.3.0'      # pin; add `browser: true` for k6/browser tests
      - run: k6 version

      - name: Run k6 test
        uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
          # flags: --once             # v2.3.0+: one-iteration smoke run of the same script
          # only-verify-scripts: true # parse with `k6 inspect` and skip execution (useful on PRs)
        env:
          BASE_URL: https://staging.example.com

      - name: Upload results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: k6-results
          path: summary.json
```

### With Grafana Cloud k6

```yaml
jobs:
  k6-cloud:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
        with:
          k6-version: '2.3.0'
      - name: Run k6 cloud test
        uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
          cloud-run-locally: true     # default: run here, stream results to Grafana Cloud k6
          cloud-comment-on-pr: true   # default: comment the test run URL on the PR
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
          K6_CLOUD_STACK_ID: ${{ secrets.K6_CLOUD_STACK_ID }}   # required since k6 v2.0.0
```

`v1.x:` `K6_CLOUD_STACK_ID` is optional (k6 falls back to the first stack with a warning). Since k6 v2.2.0 local-execution runs also stream logs to the cloud; pass `flags: --no-cloud-logs` to keep them local.

### Manual Setup (Without Official Actions)

```yaml
jobs:
  k6-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install k6
        run: |
          sudo gpg -k
          sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
          echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
          sudo apt-get update
          sudo apt-get install k6

      - name: Run load test
        run: |
          k6 version
          k6 run tests/load-test.js
        env:
          BASE_URL: ${{ vars.STAGING_URL }}
          API_KEY: ${{ secrets.API_KEY }}
```

The apt repository installs the latest stable release. To pin, download a release asset instead: `curl -sL https://github.com/grafana/k6/releases/download/v2.3.0/k6-v2.3.0-linux-amd64.tar.gz | tar xz` and move `k6-v2.3.0-linux-amd64/k6` onto the `PATH`.

### Docker-Based

```yaml
jobs:
  k6-docker:
    runs-on: ubuntu-latest
    container:
      image: grafana/k6:2.3.0     # or grafana/k6:v2 to float within the major
    steps:
      - uses: actions/checkout@v4
      - name: Run test
        run: |
          k6 version
          k6 run tests/load-test.js
```

---

## GitLab CI

### Basic Configuration

```yaml
# .gitlab-ci.yml
stages:
  - test
  - performance

k6-load-test:
  stage: performance
  image:
    name: grafana/k6:2.3.0
    entrypoint: ['']
  script:
    - k6 version
    - k6 run tests/load-test.js
  variables:
    BASE_URL: https://staging.example.com
  artifacts:
    when: always
    paths:
      - summary.json
    expire_in: 30 days
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
```

### With JSON Output

```yaml
k6-load-test:
  stage: performance
  image:
    name: grafana/k6:2.3.0
    entrypoint: ['']
  script:
    - k6 run --out json=results.json tests/load-test.js
  artifacts:
    when: always
    paths:
      - results.json
      - summary.json
```

### With Cloud Results

```yaml
k6-cloud-test:
  stage: performance
  image:
    name: grafana/k6:2.3.0
    entrypoint: ['']
  script:
    - k6 cloud run --local-execution tests/load-test.js
  variables:
    K6_CLOUD_TOKEN: $K6_CLOUD_TOKEN
    K6_CLOUD_STACK_ID: $K6_CLOUD_STACK_ID   # required since k6 v2.0.0
```

`k6 cloud run` exits `97` when the cloud run is aborted for a reason other than thresholds (v2.0.0+), so GitLab marks the job failed; `v1.x:` such aborts exit `0`.

---

## Jenkins

### Pipeline (Declarative)

```groovy
pipeline {
    agent any

    environment {
        BASE_URL = 'https://staging.example.com'
    }

    stages {
        stage('Install k6') {
            environment {
                K6_VERSION = 'v2.3.0'   // release assets are versioned: k6-v2.3.0-linux-amd64.tar.gz
            }
            steps {
                sh '''
                    curl -sL https://github.com/grafana/k6/releases/download/${K6_VERSION}/k6-${K6_VERSION}-linux-amd64.tar.gz | tar xz
                    mv k6-${K6_VERSION}-linux-amd64/k6 /usr/local/bin/
                    k6 version
                '''
            }
        }

        stage('Run Load Test') {
            steps {
                sh 'k6 run tests/load-test.js'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'summary.json', allowEmptyArchive: true
        }
    }
}
```

### Docker-Based Pipeline

```groovy
pipeline {
    agent {
        docker {
            image 'grafana/k6:2.3.0'
            args '--entrypoint=""'
        }
    }

    stages {
        stage('Run Load Test') {
            steps {
                sh 'k6 run tests/load-test.js'
            }
        }
    }
}
```

---

## Best Practices

### Separate Test Stages

Use one script and change only the invocation (k6 v2.3.0+):

```yaml
# Every PR: parse the script and run it once with its real scenario settings
smoke-test:
  script:
    - k6 inspect tests/test.js
    - k6 run --once tests/test.js

# Main branch: the full workload, or a subset of scenarios
load-test:
  script: k6 run --scenario api,checkout tests/test.js
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

`v1.x:` `k6 run --vus 3 --duration 30s tests/test.js` for the smoke stage (this replaces the script's scenarios with a default one) and an `__ENV.SCENARIO` switch for subsets.

### Environment Variables

Always pass sensitive data via CI/CD secrets:
```yaml
env:
  BASE_URL: ${{ vars.STAGING_URL }}     # Non-sensitive
  API_KEY: ${{ secrets.API_KEY }}        # Sensitive
  K6_CLOUD_TOKEN: ${{ secrets.K6_TOKEN }} # Cloud token
```

### Artifact Collection

Always collect test results regardless of pass/fail:
```yaml
- name: Upload results
  if: always()  # Runs even if test fails
  uses: actions/upload-artifact@v4
  with:
    name: k6-results
    path: |
      summary.json
      results.json
```

### Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Test completed, all thresholds passed |
| 99 | Test completed, one or more thresholds failed |
| 97 | `k6 cloud run` aborted by user, system, limit, timeout, or script error (k6 v2.0.0+). `v1.x:` these cases returned 0, so pipelines on v1 must parse the output |
| Other non-zero | Script error, setup/teardown failure, invalid options or flags (for example a v1-only flag such as `--no-summary` on k6 v2) |

### Streaming Results Without an End-of-Test Summary

```bash
k6 run --out opentelemetry --no-thresholds --summary-mode=disabled tests/test.js
```

`v1.x:` `--no-summary` is the equivalent (removed in k6 v2.0.0).
