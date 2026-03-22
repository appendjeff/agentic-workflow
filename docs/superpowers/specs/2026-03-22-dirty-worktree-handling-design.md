# Dirty Worktree Handling in `done-feature`

**Date:** 2026-03-22
**Status:** Approved

---

## Problem

`done-feature` uses `set -e` throughout. When `git worktree remove` is called on a worktree with uncommitted changes, git exits non-zero and `set -e` kills the script immediately — before branch deletion (step 5) or workspace close (step 6) can run. The user is left with:

- the worktree still on disk
- the branch still existing
- the cmux workspace still open

Recovery requires manual intervention (`git worktree remove --force`, `git branch -D`, `cmux close-workspace`).

---

## Design

Replace the bare `git worktree remove` on line 31 of `done-feature` with an `if/else` block, mirroring the existing branch-deletion pattern (lines 35–45).

### Changed section (step 4)

**Before:**
```bash
# ── 4. Remove the worktree ───────────────────────────────────────────────────
git worktree remove "$WORKTREE_DIR"
echo "Removed worktree: $WORKTREE_DIR"
```

**After:**
```bash
# ── 4. Remove the worktree ───────────────────────────────────────────────────
if git worktree remove "$WORKTREE_DIR" 2>/dev/null; then
  echo "Removed worktree: $WORKTREE_DIR"
else
  echo "Worktree has uncommitted changes:"
  git -C "$WORKTREE_DIR" status --short
  echo
  read -rp "Discard changes and force remove? [y/N] " CONFIRM
  if [[ "$CONFIRM" =~ ^[Yy]$ ]]; then
    git worktree remove --force "$WORKTREE_DIR"
    echo "Force removed worktree: $WORKTREE_DIR"
  else
    echo "Aborted. Worktree kept: $WORKTREE_DIR"
    exit 0
  fi
fi
```

### Behavior

| State | Result |
|---|---|
| Clean worktree | Identical to current behavior — no visible change |
| Dirty worktree, user confirms discard | Force removes; continues to branch deletion and workspace close |
| Dirty worktree, user declines | Prints "Aborted", exits 0; worktree, branch, and workspace all left intact. Note: the script has already `cd`'d to `$MAIN_ROOT` (step 3), so the user's shell lands in the main worktree root, not the feature worktree. |

### Key notes

- `set -e` does not apply to commands used as an `if` condition — this is standard bash behavior, no `set +e` needed.
- `2>/dev/null` suppresses git's own error output; the script prints its own message instead.
- `--force` on `git worktree remove` discards tracked and untracked changes without a separate `git clean` call.
- On abort (`exit 0`), execution never reaches branch deletion or workspace close — consistent with the user's intent to keep everything intact.

---

## Scope

Single file: `done-feature`, lines 31–32 (2 lines → ~12 lines).

No changes to `new-feature`, `session-controls`, or any other file.

---

## Verification

1. **Happy path (clean):** Run `done-feature` from a clean worktree — behavior unchanged, no prompt shown. `git worktree list` no longer shows the worktree; `git branch --list $BRANCH` returns nothing; workspace is closed.
2. **Dirty, confirm discard:** Add an uncommitted file to the worktree, run `done-feature`, enter `y` — `git worktree list` no longer shows the worktree; `git branch --list $BRANCH` returns nothing; workspace is closed.
3. **Dirty, abort:** Add an uncommitted file, run `done-feature`, enter `n` (or press Enter) — script exits 0; `git worktree list` still shows the worktree; `git branch --list $BRANCH` still returns the branch; workspace is still open; shell is in `$MAIN_ROOT`.
4. **Default-deny:** In step 3, press Enter without typing anything — confirms the prompt defaults to `N` (no force-remove).
