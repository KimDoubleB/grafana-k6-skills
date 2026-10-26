# k6 Version Compatibility Reference

Canonical version matrix and detection procedure for all skills in this plugin. Each SKILL.md repeats the three detection steps and adds its own notes; this file holds the details.

## Supported Versions

| k6 line | Latest | Status | Skill support |
|---------|--------|--------|---------------|
| **v2.x** | v2.3.0 (2026-09-21) | Current stable, receives new features | Primary target. Examples are written and checked against this version. |
| v1.x | v1.8.2 (2026-10-12) | Maintenance line, security fixes only | Supported. Differences are marked `v1.x:` in each skill. No new features land here. |
| v0.x | v0.59.0 (2025-05) | End of life | Not supported. Recommend upgrading before generating tests. |

Docker tags: `grafana/k6:latest` = newest v2.x, `grafana/k6:v2` / `grafana/k6:v1` = floating major tags, `grafana/k6:2.3.0` = exact pin, `*-with-browser` variants include Chromium.

## Detect the Target Version

Determine the version **where the test will execute**, not where the script is written. Work down this list and stop at the first answer.

| Environment | How to detect | Example |
|-------------|---------------|---------|
| Local shell / CI runner with k6 installed | `k6 version` | `k6 v2.3.0 (commit/<sha>, go<version>, <os>/<arch>)` → parse the `vMAJOR.MINOR.PATCH` token |
| Docker | Read the image tag; if floating, run the image | `docker run --rm grafana/k6:v2 version` |
| GitHub Actions | `grafana/setup-k6-action` input `k6-version` (omitted = latest stable) | `with: { k6-version: '2.3.0' }` |
| GitLab CI / Jenkins | Image tag in the job, or the install step (`apt install k6` tracks latest stable) | `image: grafana/k6:2.3.0` |
| k6-operator | `spec.runner.image` of the `TestRun`; the operator release sets the default (v1.5.0 → k6 v2.0.0, v1.6.0 → v2.2.0, v1.7.0 → v2.3.0) | `kubectl get testrun <name> -o jsonpath='{.spec.runner.image}'` |
| Grafana Cloud k6 | Cloud runners track a recent k6 release; `k6 cloud run --local-execution` uses the local binary | `k6 version` on the machine running `--local-execution` |
| Unknown | Ask the user. If no answer is possible, assume v2.x and say so. | Header comment: `// Target: k6 v2.3.0 (assumed; run 'k6 version' to confirm)` |

Always put the resolved or assumed version in a header comment of every generated script, for example:

```javascript
// Target: k6 v2.3.0 (detected via `k6 version`)
```

## Feature Availability Matrix

Use this to avoid generating syntax the target cannot run. "Fallback" is what to generate for an older target.

| Feature | Available since | Fallback on older versions |
|---------|-----------------|----------------------------|
| `--scenario name[,name]` select scenarios without editing the script | v2.3.0 | `__ENV.SCENARIO` switch inside `options.scenarios` |
| `--once` run one VU, one iteration while keeping scenario/browser options | v2.3.0 | `--vus 1 --iterations 1` (drops scenario `exec`, `env`, `tags`, browser options) |
| `async-metric-context` flag (async `group()`, tag propagation across `await`) | v2.3.0 | Keep `group()` callbacks synchronous |
| `tlsAIAFetch` option (fetch missing intermediate CA certs) | v2.3.0 | Bundle the chain on the server or use `insecureSkipTLSVerify` in non-production |
| `K6_PROMETHEUS_RW_LABELS` static labels | v2.3.0 | `--tag key=value` test-wide tags |
| `WebSocket.OPEN` and other ready-state constants | v2.3.0 | Compare `readyState` with `0`..`3` |
| `--log-ns-timestamps` | v2.3.0 | Second-precision timestamps |
| `chromium.connectOverCDP(wsEndpoint)` | v2.2.0 | `K6_BROWSER_WS_URL` environment variable |
| `handleSummaryTimeout` / `K6_HANDLE_SUMMARY_TIMEOUT` | v2.2.0 | Fixed 120 s budget; keep `handleSummary()` light |
| `TextEncoder` / `TextDecoder` globals | v2.2.0 | `k6/encoding` helpers |
| `K6_TAGS` env var; `merge-run-tags` flag | v2.2.0 | `--tag` only; tags from a higher-priority source replace lower ones |
| `--features`, `K6_FEATURES`, `k6 features` | v2.1.0 | Not available |
| `browser.newContext({ proxy })` | v2.1.0 | `K6_BROWSER_ARGS='proxy-server=host:port'` |
| `locator.isInViewport()` | v2.1.0 | Not available |
| `--vus N` alone overrides script scenarios (shared-iterations N×N) | v2.1.0 | `--vus N` silently ignored when scenarios exist |
| `k6 cloud project list`, `test list`, `load-zone list` | v2.0.0 / v2.1.0 / v2.2.0 | Grafana Cloud UI |
| Cloud secret source auto-enabled in `--local-execution` | v2.0.0 | `--secret-source=cloud` not available on v1.x (v1.8.0 enabled it, v1.8.1 reverted) |
| Subcommand extension auto-resolution (`k6 x <ext>`) | v1.7.0 | `xk6 build --with <subcommand ext>` |
| `k6 deps` dependency analysis | v1.6.0 | Read imports manually |
| JavaScript extension auto-resolution (`import 'k6/x/...'`) | v1.2.0 | `xk6 build` |
| `--summary-mode compact\|full\|disabled` | v1.x | `--no-summary` only on v1.x; removed in v2.0.0 |

## Removed or Changed in v2.0.0

Generate the v2 form by default. Only mention the v1 form when the target is v1.x.

| Area | v1.x | v2.x |
|------|------|------|
| Go module path for extensions | `go.k6.io/k6` | `go.k6.io/k6/v2` (extensions built for v1 do not load in v2 binaries) |
| Community extensions | `K6_ENABLE_COMMUNITY_EXTENSIONS=true` required | Variable removed; all catalog extensions resolve automatically |
| Disable auto-resolution | `K6_BINARY_PROVISIONING` (deprecated) | `K6_AUTO_EXTENSION_RESOLUTION=false` |
| Summary | `--no-summary`, `--summary-mode=legacy` | `--summary-mode=disabled`; `legacy` removed |
| Cloud auth | Stack optional (falls back with a warning) | Stack required: `k6 cloud login` or `K6_CLOUD_TOKEN` + `K6_CLOUD_STACK_ID` |
| Cloud exit codes | Non-threshold aborts exit `0` | Exit `97`; threshold failures stay `99` |
| `k6 cloud script.js`, `--upload-only`, `k6 login` | Deprecated but present | Removed; use `k6 cloud run`, `k6 cloud upload`, `k6 cloud login` |
| `options.ext.loadimpact` | Accepted | Removed; use `options.cloud` |
| Executor `externally-controlled` and `k6 pause/resume/scale/status` | Present | Removed, no replacement |
| REST API server | On by default at `localhost:6565` | Off unless `--address` / `K6_ADDRESS` is set |
| Redis | `k6/experimental/redis` | Removed; `import { Client } from 'k6/x/redis'` |
| OpenTelemetry output | `K6_OTEL_EXPORTER_TYPE=http\|grpc`, `K6_OTEL_SINGLE_COUNTER_FOR_RATE` | `K6_OTEL_EXPORTER_PROTOCOL=grpc\|http/protobuf`; Rate always a single counter with `condition` attribute |
| Browser Web Vitals | `browser_web_vital_fid` emitted | Removed; use `browser_web_vital_inp` |
| Legacy config path `loadimpact/config.json` | Auto-migrated | Not migrated; move to `k6/config.json` or re-run `k6 cloud login` |

## Generation Rules by Target

- **v2.x (default):** Use everything above. Prefer `--scenario` and `--once` over environment-variable switches. Never emit `K6_ENABLE_COMMUNITY_EXTENSIONS`, `--no-summary`, `externally-controlled`, or `browser_web_vital_fid`.
- **v1.8.x:** Avoid rows marked v2.1.0 and later. Keep `K6_ENABLE_COMMUNITY_EXTENSIONS=true` when a community extension is imported. Expect `browser_web_vital_fid` in output and do not threshold on it. Cloud aborts return `0`, so CI must parse the summary or use thresholds for pass/fail. The REST API is on by default.
- **v0.x:** Recommend upgrading. The v1.0.0 breaking changes are outside this plugin's scope.
- **xk6-disruptor (`testing-resilience`):** Archived upstream and pinned to the v1 Go module path. Requires a k6 v1.x binary (`grafana/k6:v1`); do not combine with v2-only features.

## Keeping This File Current

When a new k6 release appears, add its features to the matrix with the version, move anything removed to the v2 table (or a new major table), and update the version strings in README.md, CLAUDE.md, and every SKILL.md `## k6 Version Compatibility` section.
