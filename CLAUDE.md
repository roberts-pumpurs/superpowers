# Superpowers Fork

Personal fork of [obra/superpowers](https://github.com/obra/superpowers) with custom patches applied on top of upstream.

## Custom Patches

Patch files live in `patches/`. Currently applied:

- `0001-customizations-KISS-no-co-authored-by-research_codeb.patch` — adds KISS/reuse/security guidelines, no-Co-Authored-By rule for commits, routes all Rust tasks to rust-engineer, requires /research_codebase before implementation

## Syncing with Upstream

```bash
git fetch upstream
git rebase upstream/main
# If the patch conflicts during rebase, resolve it, then regenerate:
git format-patch HEAD~1 -o patches/ --force
git push origin main --force-with-lease
```

## Unapply / Reapply Patch

```bash
# Unapply (creates a revert commit)
git revert <patch-commit-sha>

# Reapply from patch file
git am patches/0001-*.patch
```

## How This Plugin Is Loaded

This repo is registered as a custom marketplace via `extraKnownMarketplaces` in `~/.claude/settings.json`:

```json
"extraKnownMarketplaces": {
  "superpowers-dev": {
    "source": {
      "source": "github",
      "repo": "roberts-pumpurs/superpowers"
    }
  }
}
```

The plugin is enabled as `superpowers@superpowers-dev` (replacing `superpowers@claude-plugins-official`).
