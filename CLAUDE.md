# Agentic Workflow Scripts

Shell scripts that pair git worktrees with cmux workspaces and Claude Code to run parallel, isolated feature development. Each feature gets its own branch, worktree, and terminal workspace — with Claude scoped only to that feature's files.

## Scripts

**`new-feature`** — Run from inside any git repo. Prompts for a feature name, creates a git worktree on a new `feature/<slug>` branch, opens a cmux workspace, starts Claude in the left pane, and opens nvim in a right split.

**`done-feature`** — Run from inside a feature worktree when work is complete. Removes the worktree, deletes the branch, and closes the cmux workspace.

## cmux CLI Reference

Run `cmux --help` to find a full reference of the cmux manual. 

