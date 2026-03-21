# Agentic Workflow

Shell scripts for running parallel AI-assisted feature development using [cmux](https://cmux.com), git worktrees, and [Claude Code](https://claude.ai/code).

## Motivation

When working with agentic AI tools like Claude Code, context isolation matters. Switching branches mid-session or sharing a working directory across multiple features leads to confused agents and tangled diffs.

These scripts create a clean, reproducible workflow:
- Each feature gets its own **git worktree** — a fully isolated checkout of the repo on a dedicated branch
- Each feature gets its own **cmux workspace** — a separate terminal tab, named after the feature
- Each workspace launches **Claude Code** autonomously, scoped only to that feature's files

The result: multiple features in flight simultaneously, each with its own AI agent, zero context bleed.

## Dependencies

- [cmux](https://cmux.com) — terminal with workspace/tab management CLI
- [Claude Code](https://claude.ai/code) — `claude` CLI available in PATH
- git 2.5+

## Installation

Clone or copy the scripts into a directory on your `$PATH`:

```bash
git clone https://github.com/appendjeff/agentic-workflow.git ~/bin
```

Add `~/bin` to your PATH if needed (e.g. in `~/.zshrc`):

```bash
export PATH="$HOME/bin:$PATH"
```

## Commands

### `new-feature`

Run from anywhere inside a git repository. Creates a new isolated workspace for a feature.

```
$ new-feature
Feature name: user auth redesign
```

**What it does:**
1. Prompts for a feature name (e.g. `"user auth redesign"`)
2. Slugifies it into a branch name: `feature/user-auth-redesign`
3. Ensures `.worktrees/` is in `.gitignore` (adds it if missing)
4. Fetches `origin` and creates a worktree branching from `origin/main`
5. Opens a new named cmux workspace pointed at the worktree
6. Starts `claude --dangerously-skip-permissions` in that workspace

**Result:** A new cmux tab named `"user auth redesign"` with Claude running in `.worktrees/user-auth-redesign/` on branch `feature/user-auth-redesign`.

---

### `done-feature`

Run from **inside** a feature worktree when the work is complete (merged or abandoned).

```
$ done-feature
Cleaning up:
  Branch:    feature/user-auth-redesign
  Worktree:  /path/to/repo/.worktrees/user-auth-redesign
  Workspace: 5B485556-0051-4F7C-826D-189F4E61F199
```

**What it does:**
1. Guards against running from outside a `.worktrees/` directory
2. Navigates to the main repo root
3. Removes the worktree directory
4. Deletes the local branch (`-d`; prompts before force-deleting if unmerged)
5. Closes the cmux workspace (last, since this kills the shell)

## Typical workflow

```
# Start a new feature from any directory inside your repo
$ new-feature
Feature name: redesign search

# ... Claude works in the isolated worktree ...
# ... you review, merge the PR ...

# Clean up from inside the worktree tab
$ done-feature
```

Multiple features can be in flight simultaneously — each in its own cmux workspace with its own Claude session.
