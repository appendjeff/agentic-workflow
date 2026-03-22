# Dirty Worktree Handling Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix `done-feature` so it gracefully handles dirty worktrees instead of dying under `set -e` and leaving partial state.

**Architecture:** Replace the bare `git worktree remove` on line 31 with an `if/else` block that mirrors the existing branch-deletion pattern (lines 35–45). Clean worktrees proceed silently; dirty ones show uncommitted files and prompt for confirmation before force-removing.

**Tech Stack:** Bash, git

---

## Files

- Modify: `done-feature:31-32`

---

### Task 1: Replace bare `git worktree remove` with guarded if/else block

**Files:**
- Modify: `done-feature:31-32`

- [ ] **Step 1: Verify the current state**

  Read lines 29–35 of `done-feature` to confirm the current code matches what the plan expects before editing:

  ```bash
  # ── 4. Remove the worktree ───────────────────────────────────────────────────
  git worktree remove "$WORKTREE_DIR"
  echo "Removed worktree: $WORKTREE_DIR"
  ```

- [ ] **Step 2: Replace lines 30–32 with the guarded block**

  Replace section 4 of `done-feature` with:

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

  Key points:
  - The `if` condition suppresses `set -e` for the first `git worktree remove` — standard bash behavior
  - `2>/dev/null` suppresses git's own error output so our message appears instead
  - `--force` on the second call discards tracked and untracked changes
  - On abort (`N` or Enter), `exit 0` prevents branch deletion and workspace close from running

- [ ] **Step 3: Manual test — clean worktree (happy path)**

  From a real feature worktree with no uncommitted changes, run `done-feature`. Expected:
  - No prompt shown
  - "Removed worktree: ..." printed
  - Continues to branch deletion and workspace close
  - `git worktree list` no longer shows the removed worktree

- [ ] **Step 4: Manual test — dirty worktree, confirm discard**

  Create a scratch worktree, add an uncommitted file, run `done-feature`, enter `y`. Expected:
  - "Worktree has uncommitted changes:" printed
  - `git status --short` output shown (e.g., `?? scratch.txt`)
  - Prompt appears: "Discard changes and force remove? [y/N]"
  - After `y`: "Force removed worktree: ..." printed
  - Continues to branch deletion
  - `git worktree list` no longer shows the removed worktree

- [ ] **Step 5: Manual test — dirty worktree, abort**

  Create a scratch worktree, add an uncommitted file, run `done-feature`, press Enter without typing anything (default No — also verify this explicitly, not just typing `n`). Expected:
  - Same status output and prompt as step 4
  - After Enter: "Aborted. Worktree kept: ..." printed, script exits
  - `git worktree list` still shows the worktree
  - `git branch --list $BRANCH` still returns the branch
  - Workspace still open
  - Shell is now in `$MAIN_ROOT` (the script `cd`'d there before the prompt)

- [ ] **Step 6: Commit**

  ```bash
  git add done-feature
  git commit -m "fix: handle dirty worktrees in done-feature with user prompt"
  ```
