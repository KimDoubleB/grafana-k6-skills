# k6 Skills for Claude Code

A [Claude Code](https://code.claude.com/docs/en/overview) plugin that provides comprehensive skills for [Grafana k6](https://k6.io/) load testing. Generate test scripts, design load scenarios, analyze results, and operate k6 infrastructure — all through natural language conversations.

## Table of Contents

- [Overview](#overview)
- [Supported k6 Versions](#supported-k6-versions)
- [Skills](#skills)
- [Installation](#installation)
- [Usage](#usage)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

## Overview

This plugin extends Claude Code with 8 specialized skills covering the full k6 load testing lifecycle. Each skill contains curated reference documentation synthesized from official k6 docs, enabling Claude to generate accurate, production-ready k6 test scripts and configurations.

**What this plugin helps you do:**

- Generate load test scripts for HTTP/REST, gRPC, and WebSocket APIs
- Create browser-based performance tests with Web Vitals measurement
- Auto-generate k6 tests from existing codebases or OpenAPI specifications
- Design load profiles with the right executors, scenarios, and thresholds
- Analyze test results, identify bottlenecks, and optimize performance
- Set up k6 in CI/CD pipelines and Kubernetes distributed environments
- Build chaos engineering tests with fault injection

### Relationship to the official `k6 x agent` skills

Since k6 v2.1, `k6 x agent init claude-code` installs Grafana's own bundled skills (test planner, load/smoke/browser test, Playwright converter) and registers the `k6 x mcp` server. This plugin complements them: it adds generation from OpenAPI specs and existing codebases, CI/CD and k6-operator operations, results and bottleneck analysis, and resilience testing. The two can be installed side by side; use `k6 x mcp`'s `validate_script` or `k6 inspect` to validate scripts generated here.

## Supported k6 Versions

| k6 line | Latest release | Status | Plugin support |
|---------|----------------|--------|----------------|
| **v2.x** | v2.3.0 (2026-09-21) | Current stable | Primary target. All examples are written and checked against this version. |
| v1.x | v1.8.2 (2026-10-12) | Maintenance (security fixes only) | Supported. Differences are marked `v1.x:` inside each skill. |
| v0.x | v0.59.0 | End of life | Not supported. Upgrade first. |

Every skill starts by determining the k6 version of the environment where the test will run (`k6 version`, the Docker image tag, the `setup-k6-action` `k6-version` input, or the k6-operator runner image), then applies the matching notes. If the version cannot be determined, the skill asks and otherwise assumes v2.x, recording the assumption in a header comment of the generated script. The full feature/version matrix lives in `plugins/grafana-k6-skills/skills/operating-k6-in-ci-cd/reference/version-compatibility.md`.

> **Note on resilience testing:** `xk6-disruptor`, which the `testing-resilience` skill is built on, was archived by Grafana Labs in February 2026 and only builds into k6 v1.x binaries. The skill documents the constraint; see its SKILL.md before use.

## Skills

### Test Generation

| Skill | Description |
|-------|-------------|
| **generating-api-load-tests** | Generate k6 scripts for HTTP/REST, gRPC, and WebSocket. Covers authentication, data parameterization, CRUD patterns, batch requests, and streaming. |
| **generating-browser-tests** | Create browser-based E2E performance tests using k6's Chromium module. Measure Core Web Vitals (LCP, FCP, CLS, INP, TTFB) and simulate user interactions. |
| **generating-tests-from-code** | Analyze existing API codebases (Express, NestJS, Spring Boot, FastAPI, Django) to auto-generate k6 tests. Detects authentication middleware and protected endpoints. |
| **generating-tests-from-openapi** | Generate k6 tests from OpenAPI 3.0/3.1 specifications. Extracts endpoints, parameters, request schemas, and security schemes to produce complete test suites. |

### Test Design & Analysis

| Skill | Description |
|-------|-------------|
| **designing-test-scenarios** | Design load profiles with the right executor (ramping-vus, constant-arrival-rate, etc.), configure multi-scenario orchestration, and set pass/fail thresholds. |
| **analyzing-test-results** | Interpret k6 output, analyze HTTP timing breakdowns, identify bottleneck patterns, and provide optimization recommendations. |

### Infrastructure & Operations

| Skill | Description |
|-------|-------------|
| **operating-k6-in-ci-cd** | Set up k6 in CI/CD pipelines (GitHub Actions, GitLab CI, Jenkins), run distributed tests with k6-operator on Kubernetes, and manage xk6 extensions. |
| **testing-resilience** | Create chaos engineering tests using xk6-disruptor. Inject faults (HTTP errors, network delays, pod termination) into Kubernetes services during load tests. |

## Installation

### Requirements

- [Claude Code](https://code.claude.com/docs/en/overview) **v2.1.295 or later** is recommended. Earlier versions clone every git submodule of a marketplace repository; this repository keeps large reference submodules under `.specs/`, so older clients may hit the clone timeout. If you must use an older version, add the marketplace with a sparse checkout (see below).
- `git` on your machine (Claude Code clones the marketplace with it).

### Install in a Claude Code session

```text
/plugin marketplace add KimDoubleB/grafana-k6-skills
/plugin install k6@grafana-k6-skills
```

The second command opens the plugin's details in the `/plugin` panel; pick a scope (user, project, or local) to finish. The marketplace is registered under the `name` in its `marketplace.json`, which is `grafana-k6-skills`, not the repository name.

### Install from your shell (scripts, CI)

```bash
claude plugin marketplace add KimDoubleB/grafana-k6-skills
claude plugin install k6@grafana-k6-skills            # add --scope project to enable it for a repository

# One-liner (Claude Code v2.1.292+): adds the marketplace and installs in one step
claude plugin install k6 --marketplace KimDoubleB/grafana-k6-skills

# Older Claude Code: fetch only the plugin directories, skipping the reference submodules
claude plugin marketplace add KimDoubleB/grafana-k6-skills --sparse .claude-plugin plugins
```

### Install for a whole repository (team)

Commit this to the repository's `.claude/settings.json`. Claude Code registers the marketplace for every collaborator who trusts the folder and enables the plugin; each collaborator then runs `claude plugin install k6@grafana-k6-skills --scope project` once to download it.

```json
{
  "extraKnownMarketplaces": {
    "grafana-k6-skills": {
      "source": { "source": "github", "repo": "KimDoubleB/grafana-k6-skills" }
    }
  },
  "enabledPlugins": {
    "k6@grafana-k6-skills": true
  }
}
```

### Verify the installation

```bash
claude plugin list          # shows k6@grafana-k6-skills with Status: ✔ enabled
claude plugin details k6    # Component inventory: Skills (8) ... and the projected token cost
```

In a session, type `/k6:` and the eight skills appear as `/k6:<skill-name>`. Skills also trigger automatically from natural-language requests (see [Usage](#usage)).

### Update

Third-party marketplaces do not auto-update by default. To pull a new release:

```bash
claude plugin marketplace update grafana-k6-skills   # refresh the catalog
claude plugin update k6@grafana-k6-skills            # update the plugin; restart or /reload-plugins to apply
```

Or, in a session, open `/plugin`, go to the **Marketplaces** tab, select `grafana-k6-skills`, and choose **Update marketplace** or **Enable auto-update**. A new copy is delivered only when the plugin's `version` changes, so releases of this plugin always bump the version in `plugin.json`.

### Uninstall

```bash
claude plugin uninstall k6@grafana-k6-skills
claude plugin marketplace remove grafana-k6-skills   # also uninstalls every plugin from this marketplace
```

### Standalone skills (without the plugin system)

Copy the skill folders into your personal skills directory:

```bash
git clone --depth 1 https://github.com/KimDoubleB/grafana-k6-skills.git /tmp/k6-skills-temp && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/k6-skills-temp/plugins/grafana-k6-skills/skills/* ~/.claude/skills/ && \
rm -rf /tmp/k6-skills-temp
```

Installed this way the skills load without the `k6:` prefix (`/generating-api-load-tests` instead of `/k6:generating-api-load-tests`), and the `/k6:<skill>` cross-references inside the skills do not resolve as commands. You also have to repeat the copy to update. Prefer the plugin install when you can.

## Usage

Skills are automatically activated when Claude detects a relevant context. You can also invoke them explicitly using the `/k6:<skill-name>` syntax. Every skill first determines the k6 version of your environment (see [Supported k6 Versions](#supported-k6-versions)) and adapts its output.

### Generate an API Load Test

```
Create a k6 load test for my REST API at https://api.example.com/users
that tests GET, POST, and DELETE endpoints with Bearer token authentication.
```

Or explicitly:

```
/k6:generating-api-load-tests

Generate a gRPC load test for the UserService with streaming support.
```

### Design Test Scenarios

```
/k6:designing-test-scenarios

I need a stress test that gradually increases from 50 to 500 virtual users
over 10 minutes, holds for 30 minutes, then ramps down.
```

### Generate Tests from Code

```
/k6:generating-tests-from-code

Analyze the Spring Boot API in this project and generate k6 load tests
for all REST endpoints, including authentication setup.
```

### Generate Tests from OpenAPI Spec

```
/k6:generating-tests-from-openapi

Generate k6 tests from the OpenAPI spec at ./api/openapi.yaml.
Include authentication setup based on the security schemes defined in the spec.
```

### Analyze Test Results

```
/k6:analyzing-test-results

Here are my k6 test results. The p95 response time jumped from 200ms to 2s
at 100 VUs. Help me identify the bottleneck.
```

### Set Up CI/CD Pipeline

```
/k6:operating-k6-in-ci-cd

Create a GitHub Actions workflow that runs k6 smoke tests on every PR
and full load tests on merge to main.
```

### Create Resilience Tests

```
/k6:testing-resilience

Create a chaos engineering test that injects 500ms latency and 10% HTTP 503 errors
into the payment-service while running a load test against the checkout flow.
```

## Project Structure

```
.
├── .claude-plugin/
│   └── marketplace.json             # Marketplace catalog
├── plugins/
│   └── grafana-k6-skills/           # Plugin (installation unit)
│       ├── .claude-plugin/
│       │   └── plugin.json          # Plugin manifest
│       └── skills/
│           ├── generating-api-load-tests/
│           │   ├── SKILL.md         # HTTP, gRPC, WebSocket test generation
│           │   └── reference/
│           ├── generating-browser-tests/
│           │   ├── SKILL.md         # Browser E2E performance tests
│           │   └── reference/
│           ├── generating-tests-from-code/
│           │   ├── SKILL.md         # Auto-generate from API source code
│           │   └── reference/
│           ├── generating-tests-from-openapi/
│           │   ├── SKILL.md         # Generate from OpenAPI 3.0/3.1 specs
│           │   └── reference/
│           ├── designing-test-scenarios/
│           │   ├── SKILL.md         # Scenario design & executor selection
│           │   └── reference/
│           ├── analyzing-test-results/
│           │   ├── SKILL.md         # Results interpretation & optimization
│           │   └── reference/
│           ├── operating-k6-in-ci-cd/
│           │   ├── SKILL.md         # CI/CD & Kubernetes operations, k6 version detection
│           │   └── reference/       # includes version-compatibility.md (canonical version matrix)
│           └── testing-resilience/
│               ├── SKILL.md         # Chaos engineering & fault injection
│               └── reference/
├── .specs/                          # Maintainer-only; not part of the installed plugin
│   ├── K6_VERSION.md                # Which k6 version the content targets and the pinned snapshots
│   └── external/                    # Git submodules (shallow): k6 source @ v2.3.0, k6-docs, claude-code-docs
├── CLAUDE.md                        # Claude Code project instructions
├── README.md
├── LICENSE
└── .gitignore
```

## Prerequisites

- **[Claude Code](https://code.claude.com/docs/en/overview)** — Anthropic's CLI tool for coding with Claude (v2.1.295 or later recommended; see [Requirements](#requirements))
- **[Grafana k6](https://k6.io/)** — Load testing tool (for running generated scripts). v2.3.0 recommended; v1.8.x supported with the noted differences
- **Kubernetes cluster** — Required only for `operating-k6-in-ci-cd` (k6-operator) and `testing-resilience` (xk6-disruptor) skills

## Contributing

Contributions are welcome! To add or improve skills:

1. Fork this repository
2. Follow the existing skill structure: `SKILL.md` + `reference/` directory
3. Keep `SKILL.md` under 500 lines — use reference files for detailed content
4. Use YAML frontmatter with `name` and `description` fields
5. Cross-check reference content accuracy against [official k6 documentation](https://grafana.com/docs/k6/latest/) and run `k6 inspect` on changed examples
6. Keep the `## k6 Version Compatibility` section of each SKILL.md current
7. Submit a pull request

### Skill Guidelines

- **SKILL.md** should provide quick-start examples and route to reference files for details
- **Reference files** must be self-contained (no external links to raw docs)
- **Description** should start with "Use when..." and describe triggering conditions
- **Naming** should use gerund format (e.g., `generating-*`, `designing-*`, `analyzing-*`)

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgments

- [Grafana k6](https://k6.io/) — The load testing tool this plugin supports
- [k6 Documentation](https://grafana.com/docs/k6/latest/) — Official documentation used as the source of truth
- [Claude Code](https://code.claude.com/docs/en/overview) — The AI coding assistant this plugin extends
- [Claude Code Skills](https://code.claude.com/docs/en/skills) and [Plugins](https://code.claude.com/docs/en/plugins/install) — Skills and plugin framework documentation
