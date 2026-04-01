# Multi-Instance Coordination

Multiple Claude Code instances may work on this repo simultaneously (e.g. terminal + Zed editor). A shared workboard at `~/projects/revealui-jv/.claude/workboard.md` tracks sessions, tasks, and file reservations.

## Identity

Agent identity is resolved automatically by `session-start.js` using a 6-tier detection cascade. The result is cached for the session in `/tmp/revealui-session-<ppid>.id`.

### Identity Taxonomy

| Identity | Detection Signal | Context |
|----------|-----------------|---------|
| `wsl-root` | `CLAUDE_AGENT_ROLE` env var (tmux `-e`) | tmux window 1 |
| `revealui-terminal` | `CLAUDE_AGENT_ROLE` env var (tmux `-e`) | tmux window 2 |
| `agent-extension` | `/proc` walk finds `zed` + no `CLAUDE_TERMINAL_CONTEXT` | Zed ACP inline (editor extension) |
| `agent-extension-2`, `-3`... | same, second+ Zed window | Multiple editor extension instances |
| `agent-edit` | `/proc` walk finds `zed` + `CLAUDE_TERMINAL_CONTEXT=zed-terminal` | Zed integrated terminal |
| `agent-edit-2`, `-3`... | same, second+ instance | Multiple editor terminals |
| `agent-system` | `WT_SESSION` present (any WT profile) | Any Windows Terminal / plain WSL terminal |
| `agent-system-2`, `-3`... | same, second+ instance | Multiple system terminals |

### Detection Cascade (7 tiers)

1. **Explicit env var** — `CLAUDE_AGENT_ROLE` set (tmux `-e`, shell export). Highest priority. Legacy values (`zed-extension`, `wsl`, `forge`, etc.) are aliased to canonical names.
2. **Tmux window mapping** — If in tmux, reads `tmux_windows` from `agent-profiles.json` to map window index → identity. Auto-set by `.tmux.conf` hook on window creation/restore.
3. **Session cache** — `/tmp/revealui-session-<ppid>.id` with timestamp. Reuses previous detection.
4. **Zed detection** — Walk `/proc` parent tree for `zed` binary. Distinguish extension vs terminal via `CLAUDE_TERMINAL_CONTEXT`. Indexes via `nextIndexedId` against workboard active rows.
5. **Windows Terminal** — `WT_SESSION` present + not in tmux + no Zed. All WT profiles map to `agent-system`. Indexes via `nextIndexedId`.
6. **CWD inference** — `~/projects/` -> `wsl-root` (existing fallback).
7. **Generic** — `agent-system-N` (replaces `terminal-N`).

### Configuration

Profile mappings are in `~/.claude/agent-profiles.json`:
- `tmux_windows`: maps tmux window index → identity (e.g., `"1": "wsl-root"`, `"2": "revealui-terminal"`)
- `wt_profiles`: maps Windows Terminal profile GUID → identity
- `default_zed_extension_name` / `default_zed_terminal_name`: override Zed auto-detection names

`.tmux.conf` has an `after-new-window` hook that auto-sets `CLAUDE_AGENT_ROLE` for configured windows (including tmux-resurrect restores).

On session start, the detected identity is logged. You can check it in the workboard Sessions table.

## Automated Lifecycle (hooks handle this)

The following happen **automatically** — do not do them manually:

- **Session start**: `session-start.js` registers your row with `(starting)` and prunes rows older than 4 hours.
- **File tracking**: `workboard-update.js` updates your `files` and `updated` columns on every Edit/Write.
- **Session end**: `stop.js` removes your row and prepends a Recent entry from your `task` description. If your `files` column is non-empty, it prints a handoff warning.

## Agent Responsibilities

1. **On first real task**: Update your `task` column — change it from `(starting)` to a short description of what you're doing. This is the description that gets written to Recent on exit.
2. **Before each task**: Re-read the workboard. Check `files` of other sessions for conflicts.
3. **During work**: Update `task` as your work evolves. The hooks keep `files` and `updated` current.
4. **After completing work**: Add a timestamped Recent entry with results. Leave `task` updated so the stop hook records it accurately.

## Stale Sessions

Rows older than 4 hours are pruned automatically by `session-start.js` on any agent's next session start. You do not need to clean up manually.

## Handoff Protocol

When you stop with incomplete work that another agent must continue:

1. **Update your `task` column** before stopping: `HANDOFF-><role>: <what's left>` (e.g. `HANDOFF->revealui-terminal: run pnpm gate after schema change`)
2. **Tag the next agent in Plans**: add `<- HANDOFF from <your-id>` next to the relevant task
3. The stop hook will print a warning and write your task description (including the `HANDOFF->` prefix) to Recent — the next agent will see it immediately on workboard read

The receiving agent should:
1. Remove the `HANDOFF->` prefix from the Plans task once picked up
2. Update their own `task` column to reflect they've taken it over

## Conflict Resolution

- File reservations are **advisory**, not locks. If you must edit a reserved file, note it in Context so the other instance sees it on next read.
- For **architectural decisions** (new packages, schema changes, API contracts), add them to Plans and wait for the other instance to acknowledge before proceeding — but only if that instance is actively working in the affected area.
- **Git conflicts** are resolved by whichever instance commits second. That instance must pull and rebase or merge before pushing.

## Master Plan Protocol

1. **On session start**: Read `~/projects/revealui-jv/docs/MASTER_PLAN.md` in full. Your work must align with the current phase.
2. **Before starting any task**: Verify the task is listed in MASTER_PLAN.md's current phase. If not listed, ask the user before proceeding.
3. **After completing any task**: Update MASTER_PLAN.md checkboxes and add a session entry to Completed Work.
4. **When discovering new work**: Add it to the appropriate phase in MASTER_PLAN.md, do not create separate plan files.
5. **When multiple agents are active**: Each agent must re-read MASTER_PLAN.md before starting new work to see if another agent has updated it.
6. **When updating MASTER_PLAN.md**: Also update the Plan Reference section in `~/projects/revealui-jv/.claude/workboard.md` with the current timestamp and your agent ID.

## Workboard Format

Keep the workboard compact. The Sessions table uses these columns:
- `id`: your detected identity (e.g. `wsl-root`, `revealui-terminal`, `agent-extension`, `agent-edit`, `agent-system`)
- `env`: environment description (e.g. `PowerShell`, `Zed/WSL`)
- `started`: ISO timestamp of session start
- `task`: short description of current work
- `files`: glob or list of files you are actively modifying
- `updated`: ISO timestamp of last workboard update

Recent entries use the format: `- [YYYY-MM-DD HH:MM] id: description`

Plans are freeform markdown subsections with the instance id in the heading.
