# Post-Implementation Cleanup Procedures

After documenting results (skip in single-agent fallback mode), the lead MUST clean up everything:

## 1. Remove Worktrees

Run each worktree removal as a **separate Bash call** (no chaining). Pre-shutdown verification already commits any uncommitted changes, so `--force` is unnecessary:
```bash
git worktree remove _workers/worker-1
```
```bash
git worktree remove _workers/worker-2
```
Repeat for each worker. If a removal fails due to uncommitted changes, commit them first via `git -C _workers/worker-N add -A && git -C _workers/worker-N commit -m "lead: salvage"`, then retry the remove.

## 2. Remove Worker Directory

Run as **separate Bash calls** (don't chain git and rm commands):
```bash
rm -rf _workers/
```
```bash
git worktree prune
```

## 3. Delete Worker Branches

Run each branch delete as a **separate Bash call**:
```bash
git branch -d <FEATURE_BRANCH>-worker-1
```
```bash
git branch -d <FEATURE_BRANCH>-worker-2
```

## 4. Sync Dependencies

```bash
# Ensure main project node_modules matches merged package.json/lock file
npm install
```

This catches any dependency changes from merged code (new imports, updated lock file entries). Fast no-op if nothing changed.

## 5. Verify Clean State

```bash
git worktree list
```

Should show only the main worktree. If stale entries remain, run `git worktree prune` again.

**Note:** `TeamDelete` was already called during the Coordination phase (after the last worker confirmed shutdown). If it wasn't called there for any reason, call it now.
