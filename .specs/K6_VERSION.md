# Reference version manifest

This file records which Grafana k6 version the skill content was written against, and
which upstream snapshots are pinned under `.specs/external/` for maintainers.
Nothing under `.specs/` is shipped with the plugin (the marketplace entry points at
`./plugins/grafana-k6-skills` only), so these pins exist purely to make the referenced
version identifiable and reproducible.

## Pinned reference snapshots

| Submodule | Upstream | Pinned at | Identifies |
|---|---|---|---|
| `.specs/external/k6` | https://github.com/grafana/k6 | tag `v2.3.0` (`e0887846`, 2026-09-21) | k6 source + `release notes/` for the target version |
| `.specs/external/k6-docs` | https://github.com/grafana/k6-docs | `151bf3ef` (2026-10-13) | Docs; authoritative folder is `docs/sources/k6/v2.3.x/` (`next/` = unreleased) |
| `.specs/external/claude-code-docs` | https://github.com/ericbuess/claude-code-docs | `81934e6d` (2026-03-02) | Claude Code docs mirror (not k6; bump separately) |

Both k6 submodules are configured `shallow = true` in `.gitmodules`.

## Skill content sync status

| Field | Value |
|---|---|
| Target k6 version for the skills | **v2.3.0** (current stable; v1.8.x is the maintenance line) |
| Content last verified against | k6 **v2.3.0** / k6-docs `151bf3ef` (`docs/sources/k6/v2.3.x/`), synced 2026-10-26. Checks run: `k6 inspect` (k6 v2.3.0 binary) on the 45 complete example scripts (25 parse clean, 20 fail only on missing data files or disabled extension auto-resolution), `node --check` on all 199 JavaScript blocks (10 intentional fragments excluded), `claude plugin validate .` |
| Change history | See the git log for this file and `CLAUDE.md` "k6 Version Scope" |

Update the "Content last verified against" row in the same change that updates skill
content, so the manifest never claims a sync that did not happen. Exception: the
`testing-resilience` skill targets k6 **v1.8.x** because xk6-disruptor (archived) only builds
against the v1 Go module path; its SKILL.md states this explicitly.

## How to check what is pinned

```bash
git submodule status                                   # gitlink SHAs (no checkout needed)
git -C .specs/external/k6 describe --tags --exact-match # prints v2.3.0 when initialized
ls .specs/external/k6-docs/docs/sources/k6/ | tail -3  # newest version folders
```

## How to bump to a new k6 release

```bash
# 1. k6 source: fetch only the release tag (shallow)
git -C .specs/external/k6 fetch --depth 1 origin tag vX.Y.Z
git -C .specs/external/k6 checkout vX.Y.Z

# 2. k6-docs: move to the commit that contains docs/sources/k6/vX.Y.x
git -C .specs/external/k6-docs fetch --depth 1 origin main
git -C .specs/external/k6-docs checkout FETCH_HEAD

# 3. Record both pins, then update this manifest and CLAUDE.md "k6 Version Scope"
git add .specs/external/k6 .specs/external/k6-docs .specs/K6_VERSION.md CLAUDE.md
```

To initialize the pins on a fresh clone without pulling full history:

```bash
git submodule update --init --depth 1 .specs/external/k6 .specs/external/k6-docs
```
