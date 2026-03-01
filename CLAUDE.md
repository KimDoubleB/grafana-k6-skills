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

### Reference Files

- Must be self-contained — synthesized knowledge
- Cross-check all technical content against official docs before writing
- Use tables, code blocks, and structured markdown for scannability

### Adding a New Skill

1. Create `plugins/grafana-k6-skills/skills/<skill-name>/SKILL.md` with frontmatter
2. Create `plugins/grafana-k6-skills/skills/<skill-name>/reference/` with supporting docs
3. Verify line count: `wc -l plugins/grafana-k6-skills/skills/<skill-name>/SKILL.md` (must be < 500)
4. Add the skill to the README.md skills table

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

Skills target k6 v1.x (current stable). Key version-specific features:
- Automatic extension resolution: v1.2.0+
- `k6/browser` module: stable in v1.x
- `k6/websockets` module: preferred over legacy `k6/ws`
- `k6/experimental/csv`: for CSV parsing (experimental)

When k6 releases breaking changes, update the affected reference files and verify code examples still work.

## Cross-Skill References

Skills reference each other via `/k6:<skill-name>` syntax in their "Related Skills" sections. When renaming or removing a skill, search all SKILL.md files for references:

```bash
grep -r "/k6:" plugins/grafana-k6-skills/skills/*/SKILL.md
```

## Content Accuracy

All reference files are synthesized from:
- k6-docs git submodule (`.specs/external/k6-docs/`)

When updating content, verify against the official k6 documentation site at https://grafana.com/docs/k6/latest/. Use WebSearch for checking current API details.

## Style Conventions

- All skill content is written in English
- Code examples use JavaScript (k6's runtime language)
- Use `constant-arrival-rate` executor for load test examples (open model, production-realistic)
- Include `check()` assertions in all generated test scripts
- Use `svc.cluster.local` DNS format for Kubernetes service URLs in examples
- Thresholds should use percentile-based metrics (`p(95)`, `p(99)`) not averages
