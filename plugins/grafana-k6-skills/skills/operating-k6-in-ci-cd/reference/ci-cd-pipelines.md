# k6 CI/CD Pipeline Reference

## GitHub Actions

### Using Official Actions

```yaml
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

      - name: Run k6 test
        uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
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
      - name: Run k6 cloud test
        uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
          cloud-run-locally: true
        env:
          K6_CLOUD_TOKEN: ${{ secrets.K6_CLOUD_TOKEN }}
```

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
        run: k6 run tests/load-test.js
        env:
          BASE_URL: ${{ vars.STAGING_URL }}
          API_KEY: ${{ secrets.API_KEY }}
```

### Docker-Based

```yaml
jobs:
  k6-docker:
    runs-on: ubuntu-latest
    container:
      image: grafana/k6:latest
    steps:
      - uses: actions/checkout@v4
      - name: Run test
        run: k6 run tests/load-test.js
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
    name: grafana/k6:latest
    entrypoint: ['']
  script:
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
    name: grafana/k6:latest
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
    name: grafana/k6:latest
    entrypoint: ['']
  script:
    - k6 cloud run --local-execution tests/load-test.js
  variables:
    K6_CLOUD_TOKEN: $K6_CLOUD_TOKEN
```

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
            steps {
                sh '''
                    curl -sL https://github.com/grafana/k6/releases/latest/download/k6-linux-amd64.tar.gz | tar xz
                    mv k6-*/k6 /usr/local/bin/
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
            image 'grafana/k6:latest'
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

```yaml
# Run smoke test on every PR
smoke-test:
  script: k6 run --vus 3 --duration 30s tests/test.js

# Run full load test on main branch only
load-test:
  script: k6 run tests/test.js
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
```

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
| Other | Test execution error |
