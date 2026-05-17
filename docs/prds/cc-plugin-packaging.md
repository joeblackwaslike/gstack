# PRD: Package gstack fork as a Claude Code Plugin

## Overview

This fork of [garrytan/gstack](https://github.com/garrytan/gstack) adds one thing: packaging all gstack skills as a Claude Code plugin under the `gstack:` namespace. Instead of installing 46 skills at the top level of `~/.claude/skills/` (where they're hard to discover and conflict with other tools), this fork installs as a single plugin and prefixes every skill with `gstack:` — so `gstack:ship`, `gstack:review`, `gstack:qa`, etc.

Everything else — the skills, the philosophy, the engineering — is unchanged from upstream. Credit goes entirely to Garry Tan and upstream contributors.

---

## Problem

The upstream `setup` script installs gstack's 46 skills as individual flat entries in `~/.claude/skills/`. This means:

- Skills like `/ship`, `/review`, `/qa` live at the top level alongside every other skill, making them hard to discover as a cohesive set
- No namespace: collisions are possible with skills from other plugins
- Update path requires re-running `./setup`, which is not discoverable
- No marketplace listing: can't install via `claude plugin install gstack@joeblackwaslike`

---

## Solution

Convert the repo into a CC plugin named `gstack`. All skills automatically get the `gstack:` prefix via the plugin system. Add a `bin/gstack` CLI for ongoing maintenance (upstream syncs, releases, marketplace updates).

---

## Files Changed

| File | Change |
|------|--------|
| `.claude-plugin/plugin.json` | New — plugin manifest |
| `skills/` | New — relative symlinks to each skill dir at root |
| `package.json` | Added `link:plugin-skills` script; wired into `build` |
| `bin/gstack` | New — CLI for update/release/install/status |
| `README.md` | Fork notice prepended |

---

## Skill Mapping (46 skills)

| Old (flat) | New (plugin) |
|------------|-------------|
| `/agent-browser` | `/gstack:agent-browser` |
| `/autoplan` | `/gstack:autoplan` |
| `/benchmark` | `/gstack:benchmark` |
| `/benchmark-models` | `/gstack:benchmark-models` |
| `/browse` | `/gstack:browse` |
| `/canary` | `/gstack:canary` |
| `/careful` | `/gstack:careful` |
| `/codex` | `/gstack:codex` |
| `/connect-chrome` | `/gstack:connect-chrome` |
| `/context-restore` | `/gstack:context-restore` |
| `/context-save` | `/gstack:context-save` |
| `/cso` | `/gstack:cso` |
| `/design-consultation` | `/gstack:design-consultation` |
| `/design-html` | `/gstack:design-html` |
| `/design-review` | `/gstack:design-review` |
| `/design-shotgun` | `/gstack:design-shotgun` |
| `/devex-review` | `/gstack:devex-review` |
| `/document-release` | `/gstack:document-release` |
| `/freeze` | `/gstack:freeze` |
| `/gstack-upgrade` | `/gstack:gstack-upgrade` |
| `/guard` | `/gstack:guard` |
| `/health` | `/gstack:health` |
| `/investigate` | `/gstack:investigate` |
| `/land-and-deploy` | `/gstack:land-and-deploy` |
| `/landing-report` | `/gstack:landing-report` |
| `/learn` | `/gstack:learn` |
| `/make-pdf` | `/gstack:make-pdf` |
| `/office-hours` | `/gstack:office-hours` |
| `/open-gstack-browser` | `/gstack:open-gstack-browser` |
| `/pair-agent` | `/gstack:pair-agent` |
| `/plan-ceo-review` | `/gstack:plan-ceo-review` |
| `/plan-design-review` | `/gstack:plan-design-review` |
| `/plan-devex-review` | `/gstack:plan-devex-review` |
| `/plan-eng-review` | `/gstack:plan-eng-review` |
| `/plan-tune` | `/gstack:plan-tune` |
| `/qa` | `/gstack:qa` |
| `/qa-only` | `/gstack:qa-only` |
| `/retro` | `/gstack:retro` |
| `/review` | `/gstack:review` |
| `/scrape` | `/gstack:scrape` |
| `/setup-browser-cookies` | `/gstack:setup-browser-cookies` |
| `/setup-deploy` | `/gstack:setup-deploy` |
| `/setup-gbrain` | `/gstack:setup-gbrain` |
| `/ship` | `/gstack:ship` |
| `/skillify` | `/gstack:skillify` |
| `/sync-gbrain` | `/gstack:sync-gbrain` |
| `/unfreeze` | `/gstack:unfreeze` |

---

## `bin/gstack` CLI

```
gstack update      — sync from garrytan/gstack, rebuild, push, release, update marketplace
gstack rebuild     — regenerate skills/ symlinks only (no git ops)
gstack status      — show version and CC plugin registration status
gstack install     — register plugin in Claude Code settings files
gstack release     — create/update GitHub release at current VERSION
gstack marketplace — update agent-marketplace entry and push
```

---

## Update & Release Runbook

Full procedure for syncing from upstream and publishing a new release. `bin/gstack update` automates all of this; the runbook is the authoritative reference.

### Prerequisites

- `gh` CLI authenticated (`gh auth status`)
- `bun` installed
- `git remote -v` shows `upstream` pointing to `git@github.com:garrytan/gstack.git`
- `~/github/joeblackwaslike/agent-marketplace` cloned locally

### 1. Sync from upstream

```bash
cd /Users/joe/github/joeblackwaslike/gstack
git fetch upstream
git merge upstream/main
```

If conflicts arise, generated SKILL.md files should take upstream's version (they're regenerated in step 2). Resolve conflicts, then continue.

### 2. Rebuild

```bash
bun run build
```

Runs: `gen:skill-docs` → binary compilation → `link:plugin-skills` (refreshes `skills/` symlinks).

### 3. Stage and commit

```bash
git add -A
git diff --cached --stat
git commit -m "chore: sync to upstream v$(cat VERSION)"
```

No-op if nothing changed.

### 4. Tag the release

```bash
VERSION="$(cat VERSION)"
git tag -f "v$VERSION"
```

### 5. Push fork + tag

```bash
git push origin main
git push origin "v$VERSION" --force
```

### 6. Create GitHub release

```bash
gh release create "v$VERSION" \
  --title "gstack $VERSION" \
  --notes "Synced from garrytan/gstack v$VERSION. Packages all gstack skills as a Claude Code plugin with gstack: namespace." \
  --target main
```

If release already exists, use `gh release edit "v$VERSION" --notes "..."` instead.

### 7. Update agent-marketplace

```bash
MKTDIR="$HOME/github/joeblackwaslike/agent-marketplace"
SEMVER="$(python3 -c "v='$VERSION'.split('.'); print('.'.join(v[:3]))")"

python3 - "$MKTDIR" "$SEMVER" <<'PYEOF'
import json, sys
mkt_dir, version = sys.argv[1], sys.argv[2]
mkt_path = f"{mkt_dir}/.claude-plugin/marketplace.json"
with open(mkt_path) as f:
    mkt = json.load(f)
for plugin in mkt["plugins"]:
    if plugin["name"] == "gstack":
        plugin["version"] = version
        print(f"Updated gstack to v{version}")
        break
else:
    mkt["plugins"].append({
        "name": "gstack",
        "description": "Garry Tan's gstack AI workflow skills, packaged as a Claude Code plugin with gstack: namespace. 46 skills including ship, review, qa, browse, investigate, office-hours, and more. Fork of garrytan/gstack.",
        "source": {"source": "url", "url": "https://github.com/joeblackwaslike/gstack.git"},
        "version": version,
        "category": "productivity",
        "keywords": ["gstack", "workflow", "skills", "ship", "review", "qa", "browse", "garrytan"]
    })
    print(f"Added gstack at v{version}")
with open(mkt_path, "w") as f:
    json.dump(mkt, f, indent=2)
    f.write("\n")
PYEOF

cd "$MKTDIR"
git add .claude-plugin/marketplace.json
git commit -m "gstack: update to v$SEMVER"
git push
```

### 8. Verify

```bash
bin/gstack status
gh release view "v$VERSION" --json tagName,name
```

### Failure recovery

| Failure point | Recovery |
|---------------|----------|
| Merge conflict | Resolve; run `bun run build` again |
| Build failure | Fix root cause; re-run `bun run build` |
| Push rejected | `git pull origin main` then retry |
| Release exists | Use `gh release edit` instead of `gh release create` |
| Marketplace push rejected | `cd ~/github/joeblackwaslike/agent-marketplace && git pull --rebase && git push` |

---

## What stays the same

The `setup` script, all SKILL.md files and `.tmpl` templates, the gen-skill-docs pipeline, the `browse`/`design` binaries, and all `bin/gstack-*` utilities are unchanged from upstream. The plugin manifest and `skills/` symlinks are the only additions.
