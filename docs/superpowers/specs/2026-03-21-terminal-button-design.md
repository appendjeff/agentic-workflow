# Terminal Session Controls — Design Spec

**Date:** 2026-03-21
**Branch:** feature/enhanced-end-session
**Status:** Approved

---

## Overview

Add a `session-controls` TUI script that launches automatically in the bottom terminal pane of every cmux workspace. It displays a styled banner and an interactive action menu, giving users a visual, keyboard-driven alternative to typing `done-feature` directly.

---

## Architecture

Three components are involved:

| Component | Change |
|---|---|
| `session-controls` | **New script.** Renders the gum TUI loop and dispatches actions. |
| `new-feature` | **Small update.** Sends `session-controls` to the bottom pane after creating it. |
| `done-feature` | **Unchanged.** Called by `session-controls` when "End Session" is selected. |

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

1. `gum style` renders the "SESSION CONTROLS" banner with a border and accent color (cyan).
2. `gum choose` renders the action list. Arrow keys navigate, Enter selects.
3. On selection, `gum confirm` prompts for confirmation before executing (prevents accidental triggers). Destructive actions (e.g., End Session) use a red confirm prompt.
4. If the user cancels the confirm, the menu re-renders and the user can make another selection.
5. If the user confirms, the action executes. After a successful action, the script exits naturally. Since `done-feature` closes the cmux workspace, the pane disappears with it.

---

## Extensibility

New actions are added by:
1. Appending a new entry to the `gum choose` list
2. Adding a matching `case` branch that implements the action

The script will include a comment block stub marking where future actions belong (Create PR, Push & Merge, Abandon, etc.). No structural changes are needed to add new buttons.

---

## Implementation Details

### `session-controls`

- **Dependency check:** exits with a friendly error message if `gum` is not installed (`brew install gum`)
- **Environment:** reads `CMUX_WORKSPACE_ID` from env (set by cmux) for passing context to called scripts
- **Loop:** `while true` — render banner → `gum choose` → `gum confirm` → execute or loop
- **Styling:** banner in cyan, confirm prompt for destructive actions in red

### `new-feature` update

After opening the bottom split and capturing `TERM_SURFACE_ID`, send `session-controls` to that surface:

```bash
cmux send --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" "session-controls" >/dev/null
cmux send-key --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" enter >/dev/null
```

This is ~2 additional lines in the existing bottom-pane block.

---

## Out of Scope (Future)

- Create PR action
- Push & Merge action
- Abandon (discard branch) action
- Mouse click support
- Horizontal button bar layout (Approach B)
