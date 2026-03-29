---
name: managing-claude-config
description: "Use when making changes to Claude Code configuration (settings.json, agents, commands, teams), the superpowers fork (skill edits, patch regeneration), or syncing config across instances. Trigger when: editing dot-claude files, updating superpowers skills/patches, syncing ~/.claude and ~/.claude-account2, refreshing plugin cache, or any mention of 'dot-claude', 'superpowers fork', 'superpowers patch', 'claude settings', 'claude agents', 'sync claude config', 'plugin cache', 'update superpowers'."
---

# Managing Claude Config and Superpowers Fork

This skill covers three workflows for managing the user's Claude Code configuration:

1. **dot-claude** — shared config repo (`settings.json`, agents, commands, teams)
2. **superpowers fork** — custom patches on top of upstream superpowers
3. **sync** — keeping all clones and caches consistent

## Architecture

```
~/.config/nix/                    # nix-darwin repo
├── dot-claude/                   # submodule → roberts-pumpurs/dot-claude
├── superpowers/                  # submodule → roberts-pumpurs/superpowers
└── CLAUDE.md

~/.claude/                        # clone of dot-claude (personal account)
~/.claude-account2/               # clone of dot-claude (work account)
```

All three dot-claude locations (`~/.claude`, `~/.claude-account2`, `~/.config/nix/dot-claude/`) track the same `origin/main`. The superpowers fork carries a single squashed patch commit on top of upstream.

## Workflow 1: Editing dot-claude

dot-claude contains: `settings.json`, `settings.local.json`, `agents/`, `commands/`, `teams/`, `agent-memory/`

Changes are typically made in whichever clone you're currently working in (`~/.claude` or `~/.claude-account2`). After making changes:

### Commit and sync

```bash
# From the clone where changes were made (e.g. ~/.claude-account2)
cd ~/.claude-account2
git add <changed-files>
git commit -m "description of change"
git push origin main

# Pull into the other clone
cd ~/.claude
git pull origin main

# Pull into the submodule
cd ~/.config/nix/dot-claude
git pull origin main
```

The order doesn't matter as long as all three end up on the same commit. Always check for uncommitted changes in the other clones before pulling — `~/.claude-account2` sometimes has local `settings.json` modifications from Claude Code itself.

### What goes where

| File | Purpose | Notes |
|------|---------|-------|
| `settings.json` | Shared settings, plugin config, permissions | Claude Code may auto-modify this — check diffs carefully |
| `settings.local.json` | Machine-local overrides | Not synced meaningfully, but tracked |
| `agents/*.md` | Custom agent definitions | Agents available across all sessions |
| `commands/*.md` | Slash commands | Available via `/command-name` |
| `teams/*.md` | Team configurations | Shared team definitions |

## Workflow 2: Editing the Superpowers Fork

The fork at `~/.config/nix/superpowers/` carries a custom patch on top of upstream. The patch file lives in `patches/` and is a `git format-patch` output.

### Making changes to patched skills

1. **Edit the skill files directly** in `skills/` (they already have the patch applied)
2. **Commit** your changes to the skill files:
   ```bash
   cd ~/.config/nix/superpowers
   git add skills/
   git commit -m "description of skill changes"
   ```
3. **Regenerate the patch** — create a temp branch from upstream, apply all customizations, and format-patch:
   ```bash
   # Find the upstream merge commit (last commit before customizations)
   git log --oneline  # identify the upstream base commit

   # Create temp branch, copy current skill state, generate patch
   git checkout -b temp-patch <upstream-base-commit>
   git checkout main -- skills/
   git add skills/
   git commit -m "customizations: brief description of all changes"
   git format-patch HEAD~1 -o patches/
   git checkout main
   git branch -D temp-patch
   ```
4. **Update the patch reference** — remove the old patch file, add the new one:
   ```bash
   git rm patches/<old-patch-file>.patch
   git add patches/<new-patch-file>.patch
   ```
5. **Update `CLAUDE.md`** in the superpowers repo if the patch description changed
6. **Commit and push**:
   ```bash
   git add CLAUDE.md patches/
   git commit -m "update patch: description of changes"
   git push origin main
   ```

### Syncing with upstream

```bash
cd ~/.config/nix/superpowers
git fetch upstream
git rebase upstream/main
# Resolve any conflicts in the patch commit, then:
git format-patch HEAD~1 -o patches/
git push origin main --force-with-lease
```

## Workflow 3: Refreshing the Plugin Cache

After pushing superpowers changes, Claude Code won't pick them up until the plugin cache is refreshed. The running session overwrites `installed_plugins.json` on exit, so CLI commands alone won't stick.

### Steps

1. **Clear the cache**:
   ```bash
   rm -rf ~/.claude/plugins/cache/superpowers-dev/
   ```
2. **Re-fetch and install**:
   ```bash
   claude plugin marketplace update superpowers-dev
   claude plugin install superpowers@superpowers-dev
   ```
3. **Restart Claude Code** — this is required. The running session will overwrite plugin state on exit.

Do the same for `~/.claude-account2` if both accounts use the fork.

## Commit Rules

- Never include `Co-Authored-By` lines in commit messages
- Use concise, descriptive commit messages
- Commit specific files by name rather than `git add -A`

## Quick Reference: Full Sync Checklist

After making changes to both dot-claude and superpowers:

1. Commit and push dot-claude changes from the working clone
2. Pull dot-claude in the other two locations
3. Commit skill edits, regenerate patch, push superpowers
4. Clear plugin cache and reinstall
5. Restart Claude Code
6. Verify all three dot-claude clones are on the same commit:
   ```bash
   for d in ~/.claude ~/.claude-account2 ~/.config/nix/dot-claude; do
     echo "$d: $(git -C "$d" log --oneline -1)"
   done
   ```
