---
name: operating-k6-in-ci-cd
description: Use when setting up k6 in CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins), configuring distributed testing with k6-operator on Kubernetes, managing k6 extensions (xk6), or checking which k6 version an environment runs and what that version supports. Use when the user wants to automate performance tests in a pipeline, run distributed load tests, pin or upgrade a k6 version, or asks whether a k6 feature exists in their version.
---

# Operating k6 in CI/CD

Configure k6 for CI/CD pipelines, distributed testing on Kubernetes, and extension management. This skill also owns the canonical k6 version matrix used by every other skill in this plugin.

## k6 Version Compatibility

Written for **k6 v2.3.0** (current stable). The v1.8.x maintenance line is supported where noted; differences are marked `v1.x:` inline. Before producing pipeline or cluster configuration, determine the version of the environment that will run the test:

1. Run `k6 version` where the test will execute (prints `k6 v2.3.0 (commit/<sha>, go<version>, <os>/<arch>)`). In CI read the pin: `setup-k6-action` input `k6-version`, the Docker tag (`grafana/k6:2.3.0`, floating `grafana/k6:v2` / `grafana/k6:v1`, `latest` = newest v2.x), or the k6-operator `spec.runner.image`.
2. If no version is visible, ask the user. Otherwise assume v2.x, pin it explicitly in the generated configuration, and add a header comment such as `# Target: k6 v2.3.0` to the pipeline file.
3. Apply the notes below when the target is v1.x. Add a `k6 version` step to every pipeline so the log records what actually ran.

**Version notes for this skill:**
- `k6 cloud run` exits `97` for non-threshold aborts (user, system, timeout, script error) since v2.0.0; threshold failures stay `99`. `v1.x:` those aborts exit `0`, so CI must not treat `0` as unconditional success.
- Every `k6 cloud` command needs a stack since v2.0.0: `k6 cloud login` once, or `K6_CLOUD_TOKEN` + `K6_CLOUD_STACK_ID` in CI. `v1.x:` token alone works with a warning.
- `K6_ENABLE_COMMUNITY_EXTENSIONS` was removed in v2.0.0; community extensions resolve automatically. `v1.x:` set it to `true` when a script imports a community `k6/x/*` module.
- Extensions must import `go.k6.io/k6/v2` to load into a v2 binary; anything built against `go.k6.io/k6` (v1) only works with `grafana/k6:v1`. Subcommand extensions (`k6 x <name>`) auto-resolve since v1.7.0.
- `k6 run --once` and `--scenario` (v2.3.0+) are the way to run smoke checks and partial suites in pipelines without separate scripts. `v1.x:` `--vus 1 --iterations 1` and an `__ENV` switch.
- k6-operator v1.5.0+ runs a **k6 v2** runner by default (v1.7.0 → k6 v2.3.0); pin `spec.runner.image` to `grafana/k6:v1` for v1-only scripts. `.spec.args` (array form) exists since operator v1.6.0; `.spec.scuttle` is deprecated since v1.5.0.

The full matrix lives in [reference/version-compatibility.md](reference/version-compatibility.md): supported lines, per-environment detection commands, feature-to-version table, and what v2.0.0 removed.

## Choose Your Path

- **Version Check** → See [reference/version-compatibility.md](reference/version-compatibility.md)
  - Detect the k6 version of a local shell, Docker image, CI job, or k6-operator runner
  - Decide which syntax to generate for v2.x vs v1.x
  - Plan a v1 → v2 upgrade (removed flags, exit codes, module path)

- **CI/CD Pipeline Setup** → See [reference/ci-cd-pipelines.md](reference/ci-cd-pipelines.md)
  - GitHub Actions, GitLab CI, Jenkins
  - Docker-based execution and version pinning
  - Threshold-based pass/fail in pipelines, smoke vs full runs with `--once` / `--scenario`

- **Distributed Testing (Kubernetes)** → See [reference/k6-operator.md](reference/k6-operator.md)
  - k6-operator installation
  - TestRun CRD configuration
  - Scaling load across pods

- **Extensions (xk6)** → See [reference/extensions.md](reference/extensions.md)
  - Automatic extension resolution (JavaScript and subcommand extensions)
  - Official and community extensions, archived ones to avoid
  - Building custom k6 binaries with xk6 for the `go.k6.io/k6/v2` module

## Quick Start: GitHub Actions

```yaml
# Target: k6 v2.3.0
name: k6 Load Test
on: [push]
jobs:
  k6:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: grafana/setup-k6-action@v1
        with:
          k6-version: '2.3.0'        # omit to get the latest stable
      - run: k6 version              # record the version in the job log
      - uses: grafana/run-k6-action@v1
        with:
          path: tests/load-test.js
```

## Quick Start: Docker

```bash
# Pin the version; `grafana/k6:latest` tracks the newest v2.x, `grafana/k6:v2` / `grafana/k6:v1` float within a major
docker run --rm -i grafana/k6:2.3.0 run - < script.js

# With local script file
docker run --rm -v $(pwd):/scripts grafana/k6:2.3.0 run /scripts/test.js

# With environment variables
docker run --rm -v $(pwd):/scripts \
  -e BASE_URL=https://staging.example.com \
  grafana/k6:2.3.0 run /scripts/test.js

# Which k6 is inside an image?
docker run --rm grafana/k6:v2 version
```

## Key Concepts

### Detect the Version Before Generating Config

| Where the test runs | Command / place to look |
|---------------------|-------------------------|
| Local or CI shell | `k6 version` |
| Docker image | `docker run --rm <image> version`, or the tag itself |
| GitHub Actions | `k6-version` input of `grafana/setup-k6-action` (unset = latest) |
| k6-operator | `kubectl get testrun <name> -o jsonpath='{.spec.runner.image}'`; unset means the operator's default runner (operator v1.7.0 → k6 v2.3.0) |
| Grafana Cloud k6 (`--local-execution`) | The local binary: `k6 version` |

### Threshold-Based Pass/Fail and Exit Codes

k6 exits with code 99 if thresholds fail. CI/CD pipelines use this to pass/fail the build:

```javascript
export const options = {
  thresholds: {
    http_req_duration: ['p(95)<500'],  // Fails pipeline if p95 > 500ms
    http_req_failed: ['rate<0.01'],    // Fails pipeline if error rate > 1%
  },
};
```

| Exit code | Meaning |
|-----------|---------|
| `0` | Finished, thresholds passed |
| `99` | Finished, at least one threshold failed |
| `97` | `k6 cloud run` aborted by user, system, limit, timeout, or script error (v2.0.0+; `v1.x:` returned `0`) |
| other non-zero | Script error, setup/teardown failure, invalid config |

### Smoke Run vs Full Run (v2.3.0+)

Keep one script and vary the invocation instead of maintaining separate smoke scripts:

```bash
k6 run --once tests/load-test.js                 # one VU, one iteration; keeps the scenario's exec/env/tags/browser options
k6 run --scenario checkout tests/load-test.js    # only the named scenario, with its configured load
k6 run --scenario api,checkout --once tests/load-test.js
```

`v1.x:` `k6 run --vus 1 --iterations 1 tests/load-test.js` (replaces the scenarios with a default one) or an `__ENV.SCENARIO` switch in the script.

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
# Both variables are required since k6 v2.0.0 (v1.x: token only, stack falls back with a warning)
export K6_CLOUD_TOKEN=$TOKEN
export K6_CLOUD_STACK_ID=$STACK_ID

# Run locally, send results (and, since v2.2.0, logs) to cloud; --no-cloud-logs keeps logs local
k6 cloud run --local-execution script.js

# Run entirely in cloud
k6 cloud run script.js

# Upload without running (replaces the removed --upload-only flag)
k6 cloud upload script.js
```

Interactive setups can run `k6 cloud login` once instead of exporting variables. `k6 cloud project list`, `k6 cloud test list`, and `k6 cloud load-zone list` (`--json`) help scripts discover IDs.

### k6's Own AI Tooling

`k6 x mcp` starts the k6 MCP server (script validation, docs lookup, test execution) and `k6 x agent init claude-code|cursor|vscode-copilot|codex-cli|opencode|cline` installs Grafana's bundled skills plus the MCP registration (k6 v2.0+). This plugin complements those skills; `k6 inspect script.js` or the MCP `validate_script` tool is a good final check for anything generated here.

## Related Skills

- For generating test scripts: `/k6:generating-api-load-tests`
- For analyzing results: `/k6:analyzing-test-results`
- For resilience testing on Kubernetes: `/k6:testing-resilience`
- For scenario selection and `--once` semantics: `/k6:designing-test-scenarios`
