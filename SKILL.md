---
name: claudecode-plan-build-orchestrator
description: End-to-end workflow skill that drives a 6-phase pipeline (problem -> fix+verify -> docs sync -> knowledge graph -> commit+PR -> issue triage) by alternating plan and build modes. Use when the user asks to "address issues", "ship it", "make a PR + close tickets", or any multi-phase coding task that spans code, tests, docs, git, and GitHub issues. Three execution modes by orchestrator: (A) harness-native inside interactive Claude Code using Plan Mode, (B) Task-tool delegation to dedicated `plan`/`build` subagents, (C) an external orchestrator (e.g. Hermes) shelling out to lean headless `claude -p` workers — pinned model, project-only settings, no MCP, explicit permission mode, streamed output. Applies karpathy-guidelines discipline (assumption surfacing, surgical changes, verifiable success criteria) at every phase.
license: Apache-2.0
compatibility: |
  Required: Claude Code (or any harness with read+write+execute tools and a Task/subagent mechanism).
  Optional companion skills (soft deps -- fallback inline if missing):
    - karpathy-guidelines       (recommended: shapes plan output)
    - coding-agents-docs-guideline (phase 2 docs sync)
    - graphify                  (phase 3 knowledge graph regen -- structural layer)
    - llm-wiki                  (phase 3 wiki ingest -- editorial layer; see references/knowledge-layer-workflow.md)
    - yeet                      (phase 4 commit + PR)
    - gh-address-comments       (phase 5 issue triage)
  Optional: the `claude` CLI (headless `-p` mode) for cross-process delegated execution.
metadata:
  author: 425degree-developers
  version: "1.0.0"
  spec: https://agentskills.io/specification
  source: https://github.com/425degree-developers/claudecode-plan-build-orchestrator
---

# Plan -> Build -> Ship Orchestrator (Claude Code)

A structured 6-phase lifecycle for taking a batch of issues, bugs, or a feature request from **problem** all the way to **fixed + tested + documented + committed + PR-opened + issues-closed** in one autonomous run. The plan/build separation maps onto Claude Code's native **Plan Mode** and **subagent** mechanism. Distilled from a real reference session ([references/issue-resolution-workflow.md](references/issue-resolution-workflow.md)).

## When to Use

Trigger when the user's request matches: "address issues in <repo>" / "fix all the bugs"; "ship it" / "get this to green and PR"; "plan, fix, test, document, and PR <feature>"; any multi-issue or multi-file task touching code, tests, docs, and git; explicit plan/build separation; any "think first, code second, then verify, document, ship" task.

**Do NOT use for:** single-line typo fixes (just edit); pure investigation with no expected code changes (use the plan-only pattern in [references/plan-build-primitives.md](references/plan-build-primitives.md)); tasks scoped to one phase only ("just commit what I have" -> invoke the relevant companion skill, e.g. `yeet`).

## The 6-Phase Lifecycle

A deterministic pipeline of **plan/build pairs**, one pair per logical concern, alternating thinking (plan) and acting (build):

```
PLAN-1 Problem analysis (root causes, assumptions, criteria) -> HARD GATE: user approval
 -> BUILD-1 Implement fixes, loop internally until verifications green
 -> PLAN-2 Docs gap matrix -> SOFT GATE -> BUILD-2 Surgical doc edits
 -> PLAN-3 Knowledge layer plan -> BUILD-3 3a: graphify regen, 3b: llm-wiki ingest
 -> PLAN-4 Commit/PR strategy -> SOFT GATE -> BUILD-4 Branch + commits + push + draft PR
 -> PLAN-5 Issue triage table -> SOFT GATE -> BUILD-5 gh issue close x N / comment x M
 -> END
```

| # | Plan output                                | Build artifacts                                | Soft-dep skill                   | Skip rule                                  |
| - | ------------------------------------------ | ---------------------------------------------- | -------------------------------- | ------------------------------------------ |
| 1 | Root-cause matrix + assumptions + criteria | Code/test/config edits; verifications green    | karpathy-guidelines              | (never skip)                               |
| 2 | Docs gap matrix with priority              | Surgical edits to existing docs                | coding-agents-docs-guideline     | no `docs/` directory                       |
| 3 | Knowledge layer update plan (graph + wiki) | Regenerated `graphify-out/` + filed/updated wiki pages | graphify, llm-wiki       | 3a: no `graphify-out/` or no graphify skill; 3b: no wiki at `$WIKI_PATH`/`~/wiki` or no llm-wiki skill |
| 4 | Commit breakdown (N focused commits)       | Branch + N commits + push + draft PR           | yeet                             | no git repo / no GitHub remote             |
| 5 | Triage table (close/comment/leave)         | `gh issue close` x N, `gh issue comment` x M   | gh-address-comments              | `gh issue list --state open` returns empty |

## Approval Gates

- **HARD gate (after PLAN-1 — the only one):** present the plan and **stop** for an explicit yes/no or multi-choice answer ("yes", "proceed", "skip X"). PLAN-1 sets scope; getting it wrong wastes the whole pipeline. In Claude Code, PLAN-1 = **Plan Mode**, and `ExitPlanMode` is the approval prompt.
- **SOFT gates (after PLAN-2, PLAN-4, PLAN-5):** present output and auto-proceed to build **unless** there is a literal open question the agent cannot resolve from context (ambiguous scope, conflicting preferences, missing credentials). If zero open questions, announce "Proceeding to BUILD-N" and continue.
- **No gates** after PLAN-3 or after any BUILD — auto-chain unconditionally; the build summary block is the handoff signal.

## Model Tiers per Phase

Match the model to the cognitive load of the phase. Reasoning-heavy diagnosis gets the strong model; execution against an approved plan does not. In Mode C pass `--model` per worker; in Mode B the bundled agents pin tiers (`plan`: opus, `build`: sonnet) — override per phase where it deviates.

| Tier | Alias | Phases | Why |
| ---- | ----- | ------ | --- |
| **Strong** | `opus` (best available) | PLAN-1 (requirements, root-cause pinpointing), PLAN-5 (issue triage against the codebase) | Defining scope and diagnosing problems — getting these wrong wastes the whole pipeline. Triage is diagnosis, not bookkeeping. |
| **General** | `sonnet` | BUILD-1/2/3 (code/doc/knowledge edits following the plan), PLAN-2/PLAN-3 (gap matrices derived from Phase-1 output) | The plan already did the thinking; execution targets explicit, verifiable criteria. |
| **Weakest** | `haiku` | PLAN-4 + BUILD-4 (commit breakdown, push, draft PR, PR monitoring), BUILD-5 (`gh issue close`/`comment`) | Pure git/gh mechanics with single-command verification. |

## Execution: Three Modes

Pick the mode by **who the orchestrator is** — not by subscription tier:

| Mode | Orchestrator                          | Delegation mechanism                          | Use when                                                        |
| ---- | ------------------------------------- | --------------------------------------------- | --------------------------------------------------------------- |
| A    | Claude Code itself (interactive)      | None — main agent alternates plan/build       | Running the pipeline inside an interactive Claude Code session   |
| B    | Claude Code itself                    | Task-tool subagents (`plan`, `build`)         | You want isolated per-phase context + tool-level RO enforcement  |
| C    | An external agent or script (e.g. **Hermes**) | Lean headless `claude -p` worker processes | The orchestrator lives outside Claude Code and shells out for coding work |

The Task tool only exists *inside* a Claude Code session. An external orchestrator like Hermes cannot dispatch Task subagents — its only delegation path is Mode C, and Mode C workers **must be configured lean** (see below) or each call inherits the user's full interactive setup (largest model, all plugins, MCP servers, ask-gated permissions) and runs many times slower than necessary.

### Mode A: Harness-native (interactive Claude Code)

The main agent itself alternates thinking and acting per phase. No subagents, no shelling out.

- **Plan mode** = read-only tools (Read, Glob, Grep, WebFetch, read-only Bash like `git status`, `gh issue list`) producing structured output; no edits, writes, or mutating commands. TodoWrite is allowed for declaring the plan. Engage Claude Code **Plan Mode** (Shift+Tab): it enforces read-only at the harness level and ends with the `ExitPlanMode` approval prompt — a perfect fit for the PLAN-1 HARD gate.
- **Build mode** = read+write+execute tools (Edit, Write, Bash) implementing the plan, with TodoWrite as the goal ledger. Exit Plan Mode to enter build.

The "mode switch" is *behavioral*; the agent announces each switch ("Switching to BUILD mode for Phase 1 implementation").

### Mode B: Subagent-delegated (Claude Code as orchestrator)

Each phase runs in a clean, isolated context window with read-only vs. read-write enforced at the tool level. Two subagents ship in [agents/](agents/): `plan` (read-only: Read, Glob, Grep, WebFetch, read-only Bash) and `build` (read+write+execute: Edit, Write, Bash). Install by copying [agents/](agents/) into `.claude/agents/` (project) or `~/.claude/agents/` (user), then **restart Claude Code** so they register (until restart, `subagent_type: "build"` won't resolve and `"plan"` may fall back to the built-in `Plan` agent). Drive the pipeline:

```
PLAN-1:  Task(subagent_type="plan",  prompt="<problem statement>. Produce the Phase-1 plan artifact.")
         -> present its output to the user -> HARD GATE -> wait for approval
BUILD-1: Task(subagent_type="build", prompt="Implement this approved plan:\n<plan>\nLoop until <success criteria> are green.")
PLAN-2:  Task(subagent_type="plan",  prompt="Analyze docs gaps given the Phase-1 changes:\n<build-1 summary>")
... and so on for each phase.
```

**Context does not persist across Task calls** — each subagent starts fresh. The orchestrator threads context: pass the prior phase's plan and/or build summary into the next subagent's prompt. The main conversation is the single source of truth; subagents are stateless workers.

### Mode C: External orchestrator -> lean headless `claude -p` workers

When the orchestrator is an external agent (e.g. Hermes) or a script, shell out to the Claude Code CLI in headless mode — but **never with a bare `claude -p "<prompt>"`**. A bare invocation inherits the user's full interactive configuration: their pinned model (often the largest/slowest), every enabled plugin and MCP server, and permission rules written for an interactive session. Headless mode has no user to answer permission prompts, so every `ask` rule silently becomes a **denial** and the worker burns minutes retrying workarounds.

The canonical lean worker invocation:

```bash
# BUILD-phase worker (writes code, runs tests)
claude -p "$(cat build1_prompt.md)" \
  --model sonnet \
  --permission-mode bypassPermissions \
  --setting-sources project \
  --strict-mcp-config \
  --output-format stream-json --verbose \
  --max-budget-usd 5 \
  > build1.ndjson 2>&1 &
```

Why each flag matters (all verified against CLI 2.1.x):

| Flag | Why |
| ---- | --- |
| `--model <tier>` | Workers otherwise inherit the user's pinned interactive model (e.g. `claude-fable-5[1m]`). Pick per phase from the [Model Tiers table](#model-tiers-per-phase): `opus` for PLAN-1/PLAN-5 diagnosis, `sonnet` for builds, `haiku` for commit/PR/close mechanics. |
| `--permission-mode bypassPermissions` | Headless runs cannot answer prompts; `ask` rules become denials. `acceptEdits` only covers file edits — Bash still asks. In a trusted sandbox, bypass; otherwise scope with `--allowedTools "Edit Write Read Glob Grep Bash(*)"`. |
| `--setting-sources project` | Skips user-level settings: no global plugin/MCP loading, no inherited model pin, no interactive `ask` rules. Project `.claude/settings.json` (if any) still applies. |
| `--strict-mcp-config` | No MCP servers unless explicitly passed via `--mcp-config`. Stops playwright/context7-style servers from spawning into every worker. |
| `--output-format stream-json --verbose` | Emits NDJSON events live. The orchestrator can `tail -f` for liveness instead of staring at a silent buffered pipe for minutes. (`stream-json` requires `--verbose` with `-p`.) |
| `--max-budget-usd <n>` | Runaway cap. Current CLI builds have **no `--max-turns`** — budget is the backstop. |
| `> file &` (background) | Build phases legitimately run for minutes. Run workers async and poll; don't block the orchestrator's foreground on a pipe. |

Capture the session id from the `init` event, and re-pass the flags on **every** resume (flags are per-invocation, not persisted with the session):

```bash
SESSION_ID=$(jq -r 'select(.type=="system" and .subtype=="init") | .session_id' build1.ndjson | head -1)

# Next phase, same session so context carries over — same lean flags again
claude -p --resume "$SESSION_ID" "$(cat plan2_prompt.md)" \
  --model sonnet --permission-mode bypassPermissions \
  --setting-sources project --strict-mcp-config \
  --output-format stream-json --verbose --max-budget-usd 5 > plan2.ndjson 2>&1 &
```

For PLAN-phase workers, swap `--permission-mode bypassPermissions` for `--permission-mode plan` (read-only enforced at the harness level).

**Prompt design for workers** (this is where most wall-clock hides):

- **One concern per call.** Don't batch three tasks into one "wave" prompt — split them so failures are isolated and progress is observable.
- **Never ask a worker to hand-type large artifacts.** A 50–100KB fixture is ~25k output tokens — minutes of pure generation on any model. Instruct the worker to *write a small generator script* and run it instead.
- **Thread context explicitly** — either `--resume` the same session, or paste the prior phase's plan/Build Summary into the prompt. Don't assume the worker knows anything.

See [references/plan-build-primitives.md](references/plan-build-primitives.md) for advanced patterns (multi-round planning, parallel worktrees, plan-only, build-only) and [references/claude-code-cli-reference.md](references/claude-code-cli-reference.md) for the full flag list.

## Per-Phase Template

### Plan-phase output schema

Each plan output MUST include:

1. **Structured table or matrix** — root causes x fixes, docs x gaps, files x commits, issues x actions. Tables over prose.
2. **Explicit assumptions** — one-line bullets prefixed `Assumption:`. If the karpathy-guidelines skill is loaded, name the section "Assumptions (per karpathy guidelines)".
3. **Success criteria** — a checklist of verifiable outcomes (test counts, file paths, exit codes, gh issue states), each checkable by a single command.
4. **Skill acknowledgements** — if a companion skill was loaded, briefly note how its principles were applied.
5. **Trailing question OR auto-proceed** — HARD gate: explicit question; SOFT gate with open question: explicit question; otherwise: announce that build proceeds.

### Build-phase behavior

Each build phase MUST: (1) re-read the plan from context (or from the prompt the orchestrator threads in); (2) materialize it as a TodoWrite list — one todo per item plus a final verification todo; (3) execute sequentially, updating statuses in real time; (4) loop on failure — diagnose, apply a targeted fix, retry; never escalate back to plan for the same scope; (5) emit a summary block — files changed, verifications passed, new state — as the handoff signal to the next phase.

## Failure Policy

Build phases loop internally. On failure: read the output -> diagnose the root cause (no need to ask the user) -> apply the minimum surgical fix -> re-run the failed command (or whole suite if the fix is broad) -> repeat until the criterion is met.

Escalate back to plan **only** when: (1) the user issues a new intent requiring fresh investigation; (2) the failure reveals the plan was fundamentally wrong (rare — surface explicitly before re-planning); (3) 3+ loops on the same failure without progress — stop and ask.

## Companion Skills (Soft Dependencies)

Load the named skill at the start of its phase via the `Skill` tool (e.g. `Skill(yeet)` at PLAN-4). If missing, fall back inline:

| Phase | Companion skill              | Fallback if missing                                |
| ----- | ---------------------------- | -------------------------------------------------- |
| 1     | karpathy-guidelines          | Inline the four karpathy principles (below)        |
| 1 (opt)| security-best-practices     | Skip unless fixes touch auth/secrets               |
| 2     | coding-agents-docs-guideline | Plain markdown edits, match existing doc style     |
| 3     | graphify (structural layer)  | Skip sub-step 3a entirely                          |
| 3     | llm-wiki (editorial layer)   | Skip sub-step 3b entirely ([references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md)) |
| 4     | yeet                         | Inline git/gh commands following yeet's rules      |
| 5     | gh-address-comments          | Inline `gh issue close` / `gh issue comment`       |

Inline karpathy principles (Phase-1 fallback): (1) **think before coding** — surface assumptions; present competing interpretations rather than picking silently; (2) **simplicity first** — minimum diff, no speculative abstractions; (3) **surgical changes** — touch only what's necessary, don't improve adjacent code, match existing style; (4) **goal-driven** — every fix has a verifiable success criterion; loop until verified.

## Skip Rules

Check at the start of each plan phase; if true, jump to the next non-skippable phase and announce the skip with a one-line reason:

- **Phase 2:** `docs/` missing OR `ls docs/` empty.
- **Phase 3:** 3a if `graphify-out/` missing OR no graphify skill; 3b if no wiki at `$WIKI_PATH` (default `~/wiki`) OR no llm-wiki skill OR no durable Phase-1 insights ([checklist](references/knowledge-layer-workflow.md)). Both skipped -> skip Phase 3 entirely.
- **Phase 4:** `git rev-parse --is-inside-work-tree` fails OR `git remote get-url origin` fails. Report to user.
- **Phase 5:** `gh issue list --state open --limit 1` empty, OR `gh auth status` fails and the user declines to authenticate.

## Pre-Flight Checks

Before PLAN-1, run once and report a one-block summary:

1. **Working directory** — `pwd`, `git rev-parse --show-toplevel`; cd to a user-provided path.
2. **Git status** — `git status -sb`; uncommitted changes become part of the PR.
3. **GitHub CLI** — `gh --version && gh auth status`; if unauthenticated, ask the user to `gh auth login` before phases 4/5.
4. **Companion skills** — which soft-dep skills are loadable? For Phase 3, probe both layers (`graphify-out/` + graphify skill; `$WIKI_PATH/SCHEMA.md` + llm-wiki skill) and cache the booleans.
5. **Test runner discovery** — find the test command (`package.json` scripts, `Makefile`, `tests/run.sh`, `pytest.ini`, ...); cache for BUILD-1.

## References

- [references/plan-build-primitives.md](references/plan-build-primitives.md) — plan/build mechanics for delegated execution (Modes B and C), advanced patterns
- [references/issue-resolution-workflow.md](references/issue-resolution-workflow.md) — the 6-phase FSM in detail, per-phase prompt shapes, failure/loop patterns, worked example (5 pairs, 11 issues, 70/70 tests, 6 commits, 1 PR)
- [references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md) — Phase 3: graphify + llm-wiki division of labor
- [references/claude-code-cli-reference.md](references/claude-code-cli-reference.md) — complete `claude` CLI + subagent reference for delegated execution
