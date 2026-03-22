# Terminal Session Controls — Design Spec

**Date:** 2026-03-21
**Branch:** feature/enhanced-end-session
**Status:** Draft

---

## Overview

Add a `session-controls` TUI script that launches automatically in the bottom terminal pane of every cmux workspace. It displays a styled banner and an interactive action menu, giving users a visual, keyboard-driven alternative to typing `done-feature` directly.

---

## Architecture

Three components are involved:

| Component | Change |
|---|---|
| `session-controls` | **New script.** Renders the gum TUI loop and dispatches actions. |
| `new-feature` | **Small update.** Sends `session-controls` to the bottom pane after creating it, injecting `CMUX_WORKSPACE_ID` into the environment. |
| `done-feature` | **Code unchanged.** Called by `session-controls` when "End Session" is selected. Known interaction: see Edge Cases. |

**Deployment requirement:** `session-controls` must be on `$PATH` in the pane's shell environment (same convention as the existing `new-feature` and `done-feature` scripts).

---

## UI/UX

The bottom pane renders the following on launch:

```
╭──────────────────────╮
│   SESSION CONTROLS   │
╰──────────────────────╯
  ▸ End Session
```

**Interaction flow:**

1. `tput clear` clears the pane, then `gum style` renders the "SESSION CONTROLS" banner with a border and accent color (cyan). The banner is rendered once before the loop begins, not re-rendered on each iteration.
2. `gum choose` renders the action list. Arrow keys navigate, Enter selects.
3. On selection, `gum confirm` prompts for confirmation before executing (prevents accidental triggers). Destructive actions (e.g., End Session) use `--selected.foreground="1"` (red) on the confirm prompt.
4. If the user cancels the confirm, the loop continues — the banner is **not** re-rendered; only `gum choose` re-runs from the top of the loop.
5. If the user confirms, the action executes.
6. **Post-action policy:** Actions that close the workspace (e.g., End Session → `done-feature`) exit the script naturally; the pane disappears with the workspace. Actions that do not close the workspace (future: Create PR, Push & Merge) loop back to `gum choose` after completion, so the menu remains available.

---

## Extensibility

New actions are added by:
1. Appending a new entry to the `gum choose` list
2. Adding a matching `case` branch that implements the action and follows the post-action policy (exit if workspace-closing, loop back otherwise)

The script will include a comment block stub marking where future actions belong (Create PR, Push & Merge, Abandon, etc.). No structural changes are needed to add new buttons.

---

## Implementation Details

### `session-controls`

- **Dependency check:** exits with a friendly error message if `gum` is not installed (`brew install gum`)
- **Environment:** reads `CMUX_WORKSPACE_ID` from env — injected by `new-feature` at launch (see below)
- **Banner:** rendered once via `gum style` with `--border rounded --border-foreground 6 --align center --padding "0 2"` (cyan border)
- **Loop structure:**
  ```
  clear + render banner (once, before loop)
  while true:
    ACTION=$(gum choose ...)
    gum confirm "$ACTION?" [--selected.foreground="1" for destructive]
    if confirmed: execute action; break or loop per post-action policy
    # cancelled: loop back to gum choose
  ```
- **Confirm styling for destructive actions:** `gum confirm --selected.foreground="1"` sets the selected button text to red (ANSI color 1)

### `new-feature` update

After opening the bottom split and capturing `TERM_SURFACE_ID`:

```bash
cmux send --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" \
  "cd '$WORKTREE_DIR' && CMUX_WORKSPACE_ID='$WORKSPACE_ID' session-controls" >/dev/null
cmux send-key --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" enter >/dev/null
```

Key points:
- `cd '$WORKTREE_DIR'` ensures the pane is in the worktree before `session-controls` runs (required by `done-feature` which uses `pwd` to validate it is inside `.worktrees/`)
- `CMUX_WORKSPACE_ID='$WORKSPACE_ID'` inline-exports the workspace ID into the pane's environment; `done-feature` reads this variable directly (via `WORKSPACE_ID="${CMUX_WORKSPACE_ID:-}"`) and uses it to call `cmux close-workspace`. Without it, `done-feature` skips the close step and tells the user to close manually.

---

## Edge Cases

**Unmerged branch on End Session:** `done-feature` contains an interactive `read -rp` prompt ("Branch not fully merged. Force delete? [y/N]"). This fires after `gum confirm` has exited, so there is no terminal conflict — a raw shell prompt will appear in the pane. This is acceptable behavior for now and is noted as a known UX rough edge. A future improvement could replace this interaction with a second `gum confirm`.

**`gum` not installed:** `session-controls` prints an install hint and exits. The bottom pane will show the error message rather than the TUI. The user can still run `done-feature` manually.

---

## Out of Scope (Future)

- Create PR action
- Push & Merge action
- Abandon (discard branch) action
- Mouse click support
- Horizontal button bar layout (Approach B)
- Replacing `done-feature`'s `read -rp` with a `gum confirm`
