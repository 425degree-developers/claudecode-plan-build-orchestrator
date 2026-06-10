# Claude Code CLI + Subagent Reference

Reference for the Claude Code primitives used in plan-build orchestration: the Task tool (in-process subagents), Plan Mode, and the `claude` headless CLI (cross-process delegation).

## Delegation primitives at a glance

| Primitive            | Scope        | Context sharing                  | Analog in opencode        |
| -------------------- | ------------ | -------------------------------- | ------------------------- |
| Task tool subagent   | In-process   | None (fresh per call)            | `opencode run --agent X`  |
| `claude -p --resume` | Cross-process| Persists via session id          | `opencode run -s <id>`    |
| `claude -p --continue` | Cross-process | Resumes most recent session    | `opencode run -c`         |
| Plan Mode (Shift+Tab)| Interactive  | Same conversation                | opencode TUI plan toggle  |

## Task tool (in-process subagents)

Dispatch a named subagent from the main agent:

```
Task(subagent_type="plan",  prompt="...")   # read-only planner (bundled in agents/plan.md)
Task(subagent_type="build", prompt="...")   # read+write+execute builder (bundled in agents/build.md)
Task(subagent_type="Explore", prompt="...") # built-in read-only search agent
```

Key properties:
- The subagent runs in its **own context window**. It returns its final message to the orchestrator; nothing else from its run is visible.
- It has **no memory** of the main conversation or prior subagents. Thread any needed context into `prompt`.
- Independent subagents can be launched in parallel (multiple Task calls in one turn). Build/write subagents should run **serially** to avoid file conflicts.

### Subagent definition format

Subagents are markdown files with YAML frontmatter, in `.claude/agents/` (project) or `~/.claude/agents/` (user):

```markdown
---
name: plan
description: When this agent should be invoked.
tools: Read, Glob, Grep, WebFetch, Bash
model: inherit
---

System prompt for the agent goes here.
```

- `tools`: comma-separated allowlist. Omit to inherit all tools. The `plan` agent restricts to read-only tools; the `build` agent gets edit/write/bash.
- `model`: `inherit` (use the session model), or a specific tier (`opus`, `sonnet`, `haiku`).

## Plan Mode

Plan Mode is Claude Code's built-in read-only investigation mode — the natural home for any PLAN-N phase in Mode A.

| Action                         | How                                              |
| ------------------------------ | ------------------------------------------------ |
| Enter Plan Mode (interactive)  | Shift+Tab to cycle to "plan mode"                |
| Enter Plan Mode (headless)     | `claude -p --permission-mode plan "..."`         |
| Present plan + request approval| `ExitPlanMode` tool (the HARD-gate prompt)       |

In Plan Mode the harness blocks edits, writes, and mutating commands, so a plan phase cannot accidentally change anything. Exiting Plan Mode (user approval) transitions to the build phase.

## `claude` headless CLI

| Command / flag                       | Description                                                        |
| ------------------------------------ | ------------------------------------------------------------------ |
| `claude -p "<prompt>"`               | Headless (print) one-shot run, then exit                           |
| `claude` (no `-p`)                   | Start an interactive session                                       |
| `--resume <session_id>`              | Resume a specific session (preserves context) — analog of `-s`     |
| `--continue` / `-c`                  | Resume the most recent session                                     |
| `--permission-mode <mode>`           | `plan`, `acceptEdits`, `bypassPermissions`, `default`              |
| `--output-format <fmt>`              | `text` (default), `json`, `stream-json`                            |
| `--model <model>`                    | Override the model for this run                                    |
| `--add-dir <path>`                   | Grant access to an additional working directory (e.g. a worktree) |
| `--allowedTools "<list>"`            | Restrict the tool set for this run                                 |
| `--dangerously-skip-permissions`    | Auto-approve all permissions (use with care)                       |

### Capturing the session id

`--output-format json` emits a JSON envelope that includes `session_id`. Capture it to resume later:

```bash
claude -p "Plan: <feature>" --permission-mode plan --output-format json > plan.json
SESSION_ID=$(jq -r '.session_id' plan.json)
claude -p --resume "$SESSION_ID" "Implement the plan."
```

### JSON output shape

`--output-format json` returns a result object containing at least: `session_id`, `result` (the final assistant text), `is_error`, and usage/cost fields. Use `--output-format stream-json` for incremental events (message deltas, tool calls) when you need to watch progress.

## Mapping from opencode

| opencode                                   | Claude Code                                                  |
| ------------------------------------------ | ------------------------------------------------------------ |
| `opencode run --agent plan "..."`          | `Task(subagent_type="plan", prompt="...")` or `claude -p --permission-mode plan "..."` |
| `opencode run --agent build -s <id> "..."` | `Task(subagent_type="build", prompt="<plan>\n...")` or `claude -p --resume <id> "..."` |
| `opencode run -c "..."`                    | `claude -p --continue "..."`                                 |
| `opencode session list`                    | session id is captured from `--output-format json`           |
| `--format json`                            | `--output-format json`                                       |
| `--dir <path>`                             | run from that cwd, or `--add-dir <path>`                     |
| opencode TUI plan/build toggle             | Plan Mode (Shift+Tab) + `ExitPlanMode`                       |

## Pitfalls

- **Task subagents are stateless** — the orchestrator must thread context between phases. Only `claude -p --resume`/`--continue` preserves session context across calls.
- **Capture `session_id` before you need it** — without it, headless runs can't be resumed.
- **Plan Mode blocks writes by design** — if a "plan" run needs to write, it's actually a build phase; exit Plan Mode first.
- **Restrict build subagents to one workdir at a time** — parallelize across worktrees, never within a shared directory.
