# k6 Skills Plugin for Claude Code

A Claude Code plugin providing 8 skills for Grafana k6 load testing. This repository is structured as a plugin marketplace. Skills are in `plugins/grafana-k6-skills/skills/`, each with a `SKILL.md` and `reference/` directory.

## Project Structure

```
.claude-plugin/marketplace.json                    # Marketplace catalog
plugins/grafana-k6-skills/                         # Plugin (installation unit)
  .claude-plugin/plugin.json                       # Plugin manifest (name: "k6", author)
  skills/                                          # 8 skill directories
    <skill-name>/
      SKILL.md                                     # Main skill file with YAML frontmatter
      reference/                                   # Detailed reference docs
.specs/external/                                   # Git submodules for k6-docs and claude-code-docs
```

## Skill Authoring Rules

### SKILL.md Format

- YAML frontmatter: only `name` and `description` fields (plus optional `context`, `agent`)
- `description` starts with "Use when..." — triggering conditions only, never summarize workflow
- Max 1024 characters for frontmatter total
- Body under 500 lines — route to `reference/` files for details
- Naming convention: gerund format (`generating-*`, `designing-*`, `analyzing-*`)
- Includes a `## k6 Version Compatibility` section right after the intro paragraph (detection steps + skill-specific `v1.x:` notes)

### Reference Files

- Must be self-contained — synthesized knowledge
- Cross-check all technical content against official docs before writing
- Use tables, code blocks, and structured markdown for scannability

### Adding a New Skill

1. Create `plugins/grafana-k6-skills/skills/<skill-name>/SKILL.md` with frontmatter
2. Add the `## k6 Version Compatibility` section (copy the detection steps from an existing skill, write the skill-specific notes)
3. Create `plugins/grafana-k6-skills/skills/<skill-name>/reference/` with supporting docs
4. Verify line count: `wc -l plugins/grafana-k6-skills/skills/<skill-name>/SKILL.md` (must be < 500)
5. Add the skill to the README.md skills table

## Common Commands

```bash
# Check all SKILL.md line counts
find plugins/grafana-k6-skills/skills -name "SKILL.md" -exec sh -c 'echo "$(wc -l < "$1") $1"' _ {} \;

# List all reference files
find plugins/grafana-k6-skills/skills -path "*/reference/*" -type f | sort

# Verify frontmatter format (should show name + description)
for f in plugins/grafana-k6-skills/skills/*/SKILL.md; do head -4 "$f"; echo "---"; done

# Verify plugin.json
cat plugins/grafana-k6-skills/.claude-plugin/plugin.json

# Validate marketplace
claude plugin validate .
```

## k6 Version Scope

Skills target **k6 v2.3.0** (current stable, v2.x line). The v1.8.x maintenance line (security fixes only) is supported where a skill marks differences with a `v1.x:` prefix. The pinned reference snapshots and the "content last verified against" record live in `.specs/K6_VERSION.md`; update that file in the same change as any content sync.

Version boundaries that skills must respect:
- v2.0.0 (2026-05): Go module path `go.k6.io/k6/v2`; removed `externally-controlled`, `k6 login`, `--no-summary`, `--summary-mode=legacy`, `options.ext.loadimpact`, `k6/experimental/redis`, `K6_ENABLE_COMMUNITY_EXTENSIONS`, `browser_web_vital_fid`; REST API off by default; `k6 cloud` requires a stack; cloud aborts exit 97
- v2.1.0: feature flags (`--features`, `K6_FEATURES`, `k6 features`), browser context `proxy`, `locator.isInViewport()`
- v2.2.0: `chromium.connectOverCDP()`, `TextEncoder`/`TextDecoder` globals, `handleSummaryTimeout`
- v2.3.0: `--scenario`, `--once`, `async-metric-context` flag, `tlsAIAFetch`, `K6_PROMETHEUS_RW_LABELS`, WebSocket ready-state constants
- v1.7.0: subcommand extension auto-resolution (`k6 x <ext>`); v1.2.0: JavaScript extension auto-resolution
- `k6/experimental/csv`, `k6/experimental/fs`, `k6/experimental/streams` remain experimental

Every SKILL.md has a `## k6 Version Compatibility` section with the same three detection steps and skill-specific notes. The canonical feature/version matrix and per-environment detection procedure are in `plugins/grafana-k6-skills/skills/operating-k6-in-ci-cd/reference/version-compatibility.md`.

When k6 releases a new version: bump the submodules (procedure in `.specs/K6_VERSION.md`), diff `docs/sources/k6/v<new>.x/` against the previous folder, update affected reference files, run `k6 inspect` on changed examples, then update the version strings in CLAUDE.md, README.md, every Version Compatibility section, `version-compatibility.md`, and `.specs/K6_VERSION.md`.

## Cross-Skill References

Skills reference each other via `/k6:<skill-name>` syntax in their "Related Skills" sections. When renaming or removing a skill, search all SKILL.md files for references:

```bash
grep -r "/k6:" plugins/grafana-k6-skills/skills/*/SKILL.md
```

## Content Accuracy

All reference files are synthesized from:
- k6-docs git submodule (`.specs/external/k6-docs/`; read `docs/sources/k6/v2.3.x/`, not `next/`, which holds unreleased content)
- k6 source git submodule (`.specs/external/k6/`, tag `v2.3.0`; use it for option defaults, error messages, and `release notes/`)

Initialize them shallowly: `git submodule update --init --depth 1 .specs/external/k6 .specs/external/k6-docs`. Nothing under `.specs/` ships with the plugin.

When updating content, verify against the official k6 documentation site at https://grafana.com/docs/k6/latest/. Use WebSearch for checking current API details. Example scripts must pass `k6 inspect <script>` with a k6 v2.3.0 binary.

## Style Conventions

- All skill content is written in English
- Code examples use JavaScript (k6's runtime language)
- Use `constant-arrival-rate` executor for load test examples (open model, production-realistic)
- Include `check()` assertions in all generated test scripts
- Use `svc.cluster.local` DNS format for Kubernetes service URLs in examples
- Thresholds should use percentile-based metrics (`p(95)`, `p(99)`) not averages
- Public demo targets use `https://quickpizza.grafana.com` endpoints (`test.k6.io` and `test-api.k6.io` redirect there)
- Generated scripts start with a header comment naming the target k6 version, for example `// Target: k6 v2.3.0`
- Mark v1.x-only behavior inline with a `v1.x:` prefix; never generate removed v1 features (`externally-controlled`, `--no-summary`, `K6_ENABLE_COMMUNITY_EXTENSIONS`, `browser_web_vital_fid`)
