# k6 Skills for Claude Code

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) plugin that provides comprehensive skills for [Grafana k6](https://k6.io/) load testing. Generate test scripts, design load scenarios, analyze results, and operate k6 infrastructure — all through natural language conversations.

## Table of Contents

- [Overview](#overview)
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

### Plugin Installation (Recommended)

Install via Claude Code's plugin marketplace for automatic updates:

```
/plugin marketplace add KimDoubleB/grafana-k6-skills
/plugin install k6@KimDoubleB-grafana-k6-skills
```

After installation, the skills are available in all your Claude Code sessions.

### Standalone Skill Installation

Extract the skill folders manually:

```bash
git clone --depth 1 https://github.com/KimDoubleB/grafana-k6-skills.git /tmp/k6-skills-temp && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/k6-skills-temp/plugins/grafana-k6-skills/skills/* ~/.claude/skills/ && \
rm -rf /tmp/k6-skills-temp
```

### Verify Installation

After installation, you can verify the plugin is loaded by asking Claude Code:

```
What k6 skills do you have available?
```

## Usage

Skills are automatically activated when Claude detects a relevant context. You can also invoke them explicitly using the `/k6:<skill-name>` syntax.

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
│           │   ├── SKILL.md         # CI/CD & Kubernetes operations
│           │   └── reference/
│           └── testing-resilience/
│               ├── SKILL.md         # Chaos engineering & fault injection
│               └── reference/
├── .specs/external/                 # Git submodules (development reference)
│   ├── claude-code-docs/
│   └── k6-docs/
├── CLAUDE.md                        # Claude Code project instructions
├── README.md
├── LICENSE
└── .gitignore
```

## Prerequisites

- **[Claude Code](https://docs.anthropic.com/en/docs/claude-code)** — Anthropic's CLI tool for coding with Claude
- **[Grafana k6](https://k6.io/)** — Load testing tool (for running generated scripts)
- **Kubernetes cluster** — Required only for `operating-k6-in-ci-cd` (k6-operator) and `testing-resilience` (xk6-disruptor) skills

## Contributing

Contributions are welcome! To add or improve skills:

1. Fork this repository
2. Follow the existing skill structure: `SKILL.md` + `reference/` directory
3. Keep `SKILL.md` under 500 lines — use reference files for detailed content
4. Use YAML frontmatter with `name` and `description` fields
5. Cross-check reference content accuracy against [official k6 documentation](https://grafana.com/docs/k6/latest/)
6. Submit a pull request

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
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) — The AI coding assistant this plugin extends
- [Claude Code Skills](https://docs.anthropic.com/en/docs/claude-code/skills) — Skills framework documentation
