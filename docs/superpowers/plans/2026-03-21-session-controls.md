# Session Controls Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `session-controls` TUI script that auto-launches in the bottom pane of every cmux workspace, displaying a styled banner and interactive action menu powered by `gum`.

**Architecture:** A new standalone `session-controls` script renders a `gum style` banner once, then loops on `gum choose` + `gum confirm` to dispatch actions. `new-feature` is updated to send this script to the bottom pane (with `cd` + `CMUX_WORKSPACE_ID` injection) instead of leaving it idle. `done-feature` is untouched.

**Tech Stack:** bash, [gum](https://github.com/charmbracelet/gum) (`brew install gum`)

---

## File Map

| File | Action | Responsibility |
|---|---|---|
| `session-controls` | **Create** | TUI banner + action loop; dispatches to `done-feature` |
| `new-feature` | **Modify** (lines 87–104) | Launch `session-controls` in bottom pane instead of leaving it idle |

---

### Task 1: Create `session-controls`

**Files:**
- Create: `session-controls`

- [ ] **Step 1: Write the script**

Create `session-controls` at the repo root with this content:

```bash
#!/usr/bin/env bash
# session-controls — TUI action menu for managing the current feature session
# Auto-launched by new-feature in the bottom pane of the cmux workspace.
set -e

# ── Dependency check ─────────────────────────────────────────────────────────
if ! command -v gum &>/dev/null; then
  echo "Error: 'gum' is not installed."
  echo "Install it with: brew install gum"
  exit 1
fi

# ── Banner (rendered once, before the loop) ───────────────────────────────────
tput clear
gum style \
  --border rounded \
  --border-foreground 6 \
  --align center \
  --padding "0 2" \
  "SESSION CONTROLS"

# ── Action loop ───────────────────────────────────────────────────────────────
# To add a new action: add it to the gum choose list AND add a matching case branch.
# IMPORTANT: always wrap gum confirm in an `if` — under set -e, a non-zero exit
# (user pressed No/Escape) would otherwise kill the script instead of looping back.
# Post-action policy: workspace-closing actions call break/exit; all others loop back.
#
# Future actions to add here: "Create PR", "Push & Merge", "Abandon"
while true; do
  ACTION=$(gum choose \
    "End Session" \
  )

  case "$ACTION" in
    "End Session")
      if gum confirm "End this session?" --selected.foreground="1"; then
        done-feature
        # done-feature calls cmux close-workspace, which terminates this shell.
        # The break below is unreachable but documents intent for readers.
        break
      fi
      ;;

    # ── Future action handlers go here ───────────────────────────────────────
    # "Create PR")
    #   ...do the work...
    #   # Non-workspace-closing: fall through to loop back to gum choose
    #   ;;
    # ─────────────────────────────────────────────────────────────────────────
  esac
done
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x session-controls
```

- [ ] **Step 3: Verify the dependency check**

Temporarily rename gum to simulate it being missing:

```bash
# In a shell where gum is on PATH, test the guard:
PATH="" ./session-controls
```

Expected output:
```
Error: 'gum' is not installed.
Install it with: brew install gum
```

Expected exit code: 1

- [ ] **Step 4: Smoke-test the TUI manually**

From inside a `.worktrees/` directory (required by `done-feature`'s guard), run:

```bash
cd /path/to/any/.worktrees/some-feature
CMUX_WORKSPACE_ID=workspace:1 /path/to/session-controls
```

Verify:
- Terminal clears
- Cyan rounded-border banner reading "SESSION CONTROLS" appears
- "End Session" is listed below it
- Arrow keys navigate (only one item currently, so no movement)
- Pressing Enter on "End Session" shows a confirm prompt
- Pressing `n` / Escape cancels and re-shows the `gum choose` menu
- Pressing `y` calls `done-feature` (which will clean up the worktree)

- [ ] **Step 5: Commit**

```bash
git add session-controls
git commit -m "feat: add session-controls TUI script"
```

---

### Task 2: Update `new-feature` to launch `session-controls` in bottom pane

**Files:**
- Modify: `new-feature` (the block starting at line 87, `# ── 9. Open terminal in a bottom split`)

- [ ] **Step 1: Replace the idle bottom-pane block**

Find this block in `new-feature` (roughly lines 87–104):

```bash
# ── 9. Open terminal in a bottom split under Claude ──────────────────────────
if [[ -n "$WORKSPACE_ID" ]] && [[ -n "$CLAUDE_PANE_ID" ]]; then
  TERM_SURFACE_ID=$(cmux new-split down --workspace "$WORKSPACE_ID" 2>/dev/null \
    | grep -oE 'surface:[0-9]+' | head -1)

  if [[ -n "$TERM_SURFACE_ID" ]]; then
    # Grow Claude's pane downward to shrink the terminal to ~20%
    cmux resize-pane --workspace "$WORKSPACE_ID" --pane "$CLAUDE_PANE_ID" -D --amount 15 >/dev/null 2>&1 \
      || true
    echo "Opened terminal in bottom pane"

    # Refocus Claude's pane
    [[ -n "$CLAUDE_PANE_ID" ]] && \
      cmux focus-pane --pane "$CLAUDE_PANE_ID" --workspace "$WORKSPACE_ID"
  else
    echo "Note: could not open terminal pane"
  fi
fi
```

Replace it with:

```bash
# ── 9. Open session-controls in a bottom split under Claude ──────────────────
if [[ -n "$WORKSPACE_ID" ]] && [[ -n "$CLAUDE_PANE_ID" ]]; then
  TERM_SURFACE_ID=$(cmux new-split down --workspace "$WORKSPACE_ID" 2>/dev/null \
    | grep -oE 'surface:[0-9]+' | head -1)

  if [[ -n "$TERM_SURFACE_ID" ]]; then
    # Grow Claude's pane downward to shrink the terminal to ~20%
    cmux resize-pane --workspace "$WORKSPACE_ID" --pane "$CLAUDE_PANE_ID" -D --amount 15 >/dev/null 2>&1 \
      || true

    # Launch session-controls in the bottom pane
    cmux send --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" \
      "cd '$WORKTREE_DIR' && CMUX_WORKSPACE_ID='$WORKSPACE_ID' session-controls" >/dev/null
    cmux send-key --workspace "$WORKSPACE_ID" --surface "$TERM_SURFACE_ID" enter >/dev/null
    echo "Opened session-controls in bottom pane"

    # Refocus Claude's pane
    [[ -n "$CLAUDE_PANE_ID" ]] && \
      cmux focus-pane --pane "$CLAUDE_PANE_ID" --workspace "$WORKSPACE_ID"
  else
    echo "Note: could not open session-controls pane"
  fi
fi
```

- [ ] **Step 2: Verify the diff looks correct**

```bash
git diff new-feature
```

Confirm:
- The `cmux send` + `cmux send-key` lines for `session-controls` are present
- The `cd '$WORKTREE_DIR' && CMUX_WORKSPACE_ID=...` prefix is on the send command
- The `echo "Opened terminal in bottom pane"` message is updated
- No other lines changed

- [ ] **Step 3: End-to-end test**

From your main repo directory (not inside a worktree), run:

```bash
./new-feature
```

Enter a test feature name (e.g., `test-session-controls`). Verify:
- Workspace opens with Claude in the left pane, nvim on the right
- Bottom pane shows the `SESSION CONTROLS` banner with "End Session" option
- Selecting "End Session" and confirming cleans up the worktree, branch, and workspace

- [ ] **Step 4: Commit**

```bash
git add new-feature
git commit -m "feat: launch session-controls in bottom pane on workspace creation"
```
