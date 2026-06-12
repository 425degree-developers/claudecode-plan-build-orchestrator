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
| `--permission-mode <mode>`           | `plan`, `acceptEdits`, `bypassPermissions`, `default`, `dontAsk`, `auto` |
| `--output-format <fmt>`              | `text` (default), `json` (buffered single result), `stream-json` (live NDJSON; requires `--verbose` with `-p`) |
| `--model <model>`                    | Pin the model for this run (`sonnet`, `haiku`, `opus`, or full id). **Without it, workers inherit the user's interactive model pin.** Tier per phase: `opus` = PLAN-1/PLAN-5 diagnosis; `sonnet` = builds; `haiku` = commit/PR/close mechanics (see SKILL.md "Model Tiers per Phase"). |
| `--setting-sources <list>`           | Which settings to load: `user`, `project`, `local`. `project` alone keeps workers free of user-level plugins/model pins/ask rules. |
| `--strict-mcp-config`                | Only load MCP servers passed via `--mcp-config`; ignore all configured ones |
| `--max-budget-usd <n>`               | Hard cost cap for the run (`-p` only). Current builds have **no `--max-turns`** — this is the runaway backstop. |
| `--effort <level>`                   | `low`/`medium`/`high`/`xhigh`/`max` — lower effort = faster, cheaper turns for mechanical work |
| `--fallback-model <model>`           | Auto-fallback when the primary model is overloaded (`-p` only)     |
| `--add-dir <path>`                   | Grant access to an additional working directory (e.g. a worktree) |
| `--allowedTools "<list>"`            | Pre-approve specific tools (e.g. `"Edit Write Bash(git *)"`)       |
| `--dangerously-skip-permissions`     | Auto-approve all permissions (trusted sandboxes only)              |
| `--no-session-persistence`           | Don't save the session (`-p` only) — for fire-and-forget workers that will never be resumed |

### The lean-worker rule (read this first)

A bare `claude -p "<prompt>"` inherits the **entire interactive configuration**: the user's pinned model, every enabled plugin, every configured MCP server (each one spawns at startup), hooks, and permission rules written for a human-in-the-loop. In headless mode there is no human, so every `ask` permission rule resolves to a **denial** — the worker then wastes turns attempting workarounds. Always pin model + permissions + settings explicitly:

```bash
claude -p "<prompt>" \
  --model sonnet \
  --permission-mode bypassPermissions \
  --setting-sources project \
  --strict-mcp-config \
  --output-format stream-json --verbose \
  --max-budget-usd 5 > run.ndjson 2>&1 &
```

Use `--permission-mode plan` for plan-phase workers (harness-enforced read-only). If `bypassPermissions` is too broad for the environment, scope with `--allowedTools "Edit Write Read Glob Grep Bash(*)"` instead.

### Capturing the session id

With `stream-json`, the first `init` event carries `session_id`; the final `result` event carries the result text:

```bash
SESSION_ID=$(jq -r 'select(.type=="system" and .subtype=="init") | .session_id' run.ndjson | head -1)
RESULT=$(jq -r 'select(.type=="result") | .result' run.ndjson)
claude -p --resume "$SESSION_ID" "Implement the plan." <same lean flags>
```

With buffered `--output-format json`, the single result object has `session_id`, `result`, `is_error`, and usage/cost fields — but you see nothing until the run finishes, which can be minutes. Prefer `stream-json` whenever an orchestrator needs liveness.

Flags are **per-invocation**: a resumed session does not remember the model, permission mode, or setting sources from the previous call. Re-pass the full lean flag set on every `--resume`.

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

- **Bare `claude -p` is slow by inheritance** — no `--model` means the user's interactive model pin (possibly the largest available); no `--setting-sources`/`--strict-mcp-config` means every plugin and MCP server loads into every worker; no `--permission-mode` means `ask` rules become silent denials. This combination alone can turn a 2-minute build into a 7-minute one.
- **Headless cannot answer permission prompts** — any tool gated by an `ask` rule is denied. `acceptEdits` covers file edits only; Bash still asks. Use `bypassPermissions` (sandbox) or an explicit `--allowedTools` list.
- **Don't block on a buffered pipe** — `--output-format json` emits nothing until the run ends. Orchestrators should run workers in the background with `stream-json` and poll the NDJSON for liveness.
- **Don't ask a worker to hand-type large artifacts** — generated fixtures/datasets are pure output tokens (~minutes per 100KB). Have the worker write and run a generator script.
- **Flags are per-invocation** — `--resume` restores conversation context, not flags. Re-pass the lean flag set every call.
- **Task subagents are stateless** — the orchestrator must thread context between phases. Only `claude -p --resume`/`--continue` preserves session context across calls.
- **Capture `session_id` before you need it** — without it, headless runs can't be resumed.
- **Plan Mode blocks writes by design** — if a "plan" run needs to write, it's actually a build phase; exit Plan Mode first.
- **Restrict build subagents to one workdir at a time** — parallelize across worktrees, never within a shared directory.
