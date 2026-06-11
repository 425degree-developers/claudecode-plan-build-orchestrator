---
name: claudecode-plan-build-orchestrator
description: End-to-end workflow skill that drives a 6-phase pipeline (problem -> fix+verify -> docs sync -> knowledge graph -> commit+PR -> issue triage) by alternating plan and build modes. Use when the user asks to "address issues", "ship it", "make a PR + close tickets", or any multi-phase coding task that spans code, tests, docs, git, and GitHub issues. Built for Claude Code: runs harness-natively using Plan Mode + the standard tool set, and can delegate per phase to dedicated `plan` and `build` subagents via the Task tool (or shell out to `claude -p --resume`). Applies karpathy-guidelines discipline (assumption surfacing, surgical changes, verifiable success criteria) at every phase.
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

A structured 6-phase lifecycle for taking a batch of issues, bugs, or a feature request from **problem** all the way to **fixed + tested + documented + committed + PR-opened + issues-closed** in one autonomous run.

Built for Claude Code. The plan/build separation maps directly onto Claude Code's native **Plan Mode** and its **subagent** mechanism, so the workflow feels first-class rather than bolted on.

Distilled from a real reference session (see [references/issue-resolution-workflow.md](references/issue-resolution-workflow.md) for the worked example).

## When to Use

Trigger this skill when the user's request matches any of:

- "Address issues in <repo>" / "Fix all the bugs in <tracker>"
- "Ship it" / "Make it release-ready" / "Get this to green and PR"
- "Plan, fix, test, document, and PR <feature>"
- A multi-issue or multi-file task that touches code, tests, docs, and git
- Explicit user request for plan/build separation
- Any task where "think first, code second, then verify, document, and ship" applies

**Do NOT use for:**
- Single-line typo fixes -> just edit
- Pure investigation with no expectation of code changes -> use the plan-only pattern in [references/plan-build-primitives.md](references/plan-build-primitives.md)
- Tasks explicitly scoped to one phase only ("just commit what I have") -> invoke the relevant companion skill directly (e.g. `yeet`)

## The 6-Phase Lifecycle

The workflow is a deterministic pipeline of **plan/build pairs**, one pair per logical concern. The agent alternates thinking (plan) and acting (build) modes.

```
                                START
                                  |
                                  v
                          +---------------+
                 PHASE 1  |   PLAN-1      |  Problem analysis + fix plan
                          |   Problem     |  -> table of root causes + fixes
                          |               |  -> explicit assumptions
                          |               |  -> success criteria
                          |               |  -> HARD APPROVAL GATE
                          +-------+-------+
                                  |  user: "yes" / "proceed"
                                  v
                          +---------------+
                          |   BUILD-1     |  Implement all fixes
                          |   Fix+Verify  |  loop internally until
                          |               |  tests/verifications green
                          +-------+-------+
                                  |  (auto-chain)
                                  v
                          +---------------+
                 PHASE 2  |   PLAN-2      |  Docs gap analysis
                          |   Docs Gap    |  -> matrix of docs x gaps
                          +-------+-------+
                                  |  (auto-chain unless open question)
                                  v
                          +---------------+
                          |   BUILD-2     |  Surgical doc edits
                          |   Apply Docs  |
                          +-------+-------+
                                  |  (auto-chain)
                                  v
                          +---------------+
                 PHASE 3  |   PLAN-3      |  Knowledge layer update plan
                          |   Knowledge   |  -> graphify regen scope (3a)
                          |   Layer Plan  |  -> llm-wiki ingest scope (3b)
                          |               |  (SKIP 3a if no graphify-out/
                          |               |   OR graphify skill missing)
                          |               |  (SKIP 3b if no wiki at WIKI_PATH
                          |               |   OR llm-wiki skill missing)
                          +-------+-------+
                                  |  (auto-chain)
                                  v
                          +---------------+
                          |   BUILD-3     |  3a: run graphify regen
                          |   Knowledge   |      -> graphify-out/* regenerated
                          |   Layer Build |  3b: llm-wiki ingest (if durable
                          |               |      insights from Phase 1)
                          |               |      -> wiki pages filed/updated
                          +-------+-------+
                                  |  (auto-chain)
                                  v
                          +---------------+
                 PHASE 4  |   PLAN-4      |  Commit/PR strategy
                          |   Commit Plan |  -> N-commit breakdown
                          |               |  -> file -> issue mapping
                          +-------+-------+
                                  |  (auto-chain unless open question)
                                  v
                          +---------------+
                          |   BUILD-4     |  git checkout -b, commits,
                          |   Push + PR   |  push, gh pr create --draft
                          +-------+-------+
                                  |  (auto-chain)
                                  v
                          +---------------+
                 PHASE 5  |   PLAN-5      |  Issue triage plan
                          |   Triage      |  -> status table per issue
                          |               |  -> close/comment/leave decision
                          |               |  (SKIP if no GH issues)
                          +-------+-------+
                                  |  (auto-chain unless open question)
                                  v
                          +---------------+
                          |   BUILD-5     |  gh issue close x N
                          |   Close+Comm  |  gh issue comment x M
                          +-------+-------+
                                  |
                                  v
                                END
```

### Phase summary table

| # | Plan output                                | Build artifacts                                | Soft-dep skill                   | Skip rule                                  |
| - | ------------------------------------------ | ---------------------------------------------- | -------------------------------- | ------------------------------------------ |
| 1 | Root-cause matrix + assumptions + criteria | Code/test/config edits; verifications green    | karpathy-guidelines              | (never skip)                               |
| 2 | Docs gap matrix with priority              | Surgical edits to existing docs                | coding-agents-docs-guideline     | no `docs/` directory                       |
| 3 | Knowledge layer update plan (graph + wiki) | Regenerated `graphify-out/` artifacts + filed/updated wiki pages for durable insights | graphify, llm-wiki | graph sub-step: no `graphify-out/` or no graphify skill. wiki sub-step: no wiki at `$WIKI_PATH`/`~/wiki` or no llm-wiki skill |
| 4 | Commit breakdown (N focused commits)       | Branch + N commits + push + draft PR           | yeet                             | no git repo / no GitHub remote             |
| 5 | Triage table (close/comment/leave)         | `gh issue close` x N, `gh issue comment` x M   | gh-address-comments              | `gh issue list --state open` returns empty |

## Approval Gates

There is exactly **one HARD approval gate** at the end of PLAN-1. All other phases auto-chain.

### HARD gate (after PLAN-1)

- The agent presents the plan and **stops** with an explicit yes/no or multi-choice question.
- The agent must wait for a user response (e.g. "yes", "proceed", "skip X", "also handle Y").
- Rationale: PLAN-1 establishes scope and approach -- getting it wrong wastes the entire downstream pipeline.
- In Claude Code, PLAN-1 naturally corresponds to **Plan Mode**: stay in Plan Mode through the analysis, then present the plan via `ExitPlanMode` (which itself is the approval prompt).

### SOFT gates (after PLAN-2, PLAN-4, PLAN-5)

- The plan agent presents its output and **auto-proceeds to build** unless it has a literal open question that requires user input.
- An open question is one the agent cannot resolve from conversation context -- e.g. ambiguous scope, conflicting user preferences, missing credentials.
- If the plan surfaces zero open questions, the agent transitions to build mode immediately and announces the transition ("Proceeding to BUILD-N").

### No gates (after PLAN-3, after every BUILD)

- Auto-chain unconditionally. The build agent's summary block acts as the handoff signal.

## Execution: Two Modes

This skill works two ways inside Claude Code. Pick based on whether you have an active Claude Code subscription.

### Mode B: Subagent-delegated / headless CLI (Default for Subscribed Users)

When you have a paid Claude Code subscription, always prioritize **Mode B** to leverage the full, native execution capabilities of the `claude` CLI. It runs with complete tool permissions and loops until tests pass.

- **In-process subagents**: Delegate each phase to dedicated `plan` and `build` subagents via the **Task tool**.
- **Cross-process headless delegation**: Shell out to the `claude` CLI using `claude -p --resume <session_id>` or `--continue` to preserve session persistence across processes.

### Mode A: Harness-native (Fallback for non-subscribed/unauthenticated environments)

The main agent itself alternates thinking and acting per phase. No subagents, no shelling out.

- **Plan mode** = the agent uses read-only tools (Read, Glob, Grep, WebFetch, and read-only Bash like `git status`, `gh issue list`) and produces structured output. It does NOT edit, write, or run write commands. It may use TodoWrite to declare its plan. In Claude Code, engage **Plan Mode** (Shift+Tab) for this; Plan Mode enforces read-only at the harness level and ends with an `ExitPlanMode` approval prompt -- a perfect fit for the PLAN-1 HARD gate.
- **Build mode** = the agent uses read+write+execute tools (Edit, Write, Bash) to implement the plan. It may continue using TodoWrite as a goal ledger. Exit Plan Mode to enter build.

The "mode switch" is a *behavioral* switch the agent performs itself. The agent announces each switch ("Switching to BUILD mode for Phase 1 implementation").

### Mode B: Subagent-delegated (optional)

When you want each phase to run with a clean, isolated context window (and to enforce read-only vs. read-write at the tool level), delegate each phase to a dedicated subagent via the **Task tool**.

Two subagents ship with this skill in [agents/](agents/):

- `plan` -- read-only tools only (Read, Glob, Grep, WebFetch, read-only Bash). Cannot edit, write, or run mutating commands. Produces the structured plan artifact.
- `build` -- full read+write+execute tools (Edit, Write, Bash). Implements the plan and loops on verification failures.

Install them by copying [agents/](agents/) into `.claude/agents/` (project scope) or `~/.claude/agents/` (user scope), then **restart Claude Code** so the new agents register (until restart, `subagent_type: "build"` won't resolve and `"plan"` may fall back to the built-in `Plan` agent). Then drive the pipeline by dispatching them:

```
PLAN-1:  Task(subagent_type="plan",  prompt="<problem statement>. Produce the Phase-1 plan artifact.")
         -> present its output to the user -> HARD GATE -> wait for approval
BUILD-1: Task(subagent_type="build", prompt="Implement this approved plan:\n<plan>\nLoop until <success criteria> are green.")
PLAN-2:  Task(subagent_type="plan",  prompt="Analyze docs gaps given the Phase-1 changes:\n<build-1 summary>")
... and so on for each phase.
```

Unlike opencode's `-s <session_id>` flag, **context does not persist across Task calls** -- each subagent starts fresh. So the orchestrator (the main agent) is responsible for threading context: pass the prior phase's plan and/or build summary into the next subagent's prompt. The main conversation is the single source of truth; the subagents are stateless workers.

### Mode B (alternative): `claude -p` headless delegation

If you need cross-process delegation (e.g. the orchestrator is itself an external script), shell out to the Claude Code CLI in headless mode. This is the literal analog of opencode's `opencode run -s`:

```bash
# Phase 1: Plan (capture the session id for resumption)
claude -p "Plan: <feature>. Produce the Phase-1 plan artifact." \
  --permission-mode plan --output-format json > plan1.json
SESSION_ID=$(jq -r '.session_id' plan1.json)

# Phase 1: Build (resume the SAME session so plan context carries over)
claude -p --resume "$SESSION_ID" "Implement the approved plan. Loop until tests pass."

# Phase 2: Plan (still the same session)
claude -p --resume "$SESSION_ID" "Check gaps in docs and plan surgical updates."

# ... and so on for each phase
```

**Always pass `--resume <SESSION_ID>`** after the first command -- without it, each `claude -p` starts a fresh session and you lose the plan context. See [references/plan-build-primitives.md](references/plan-build-primitives.md) for the complete reference and advanced patterns (multi-round planning, parallel worktrees, plan-only, build-only) and [references/claude-code-cli-reference.md](references/claude-code-cli-reference.md) for the full flag list.

## Per-Phase Template

Every plan/build pair follows the same shape.

### Plan-phase output schema

Each plan output MUST include:

1. **Structured table or matrix** -- root causes x fixes, docs x gaps, files x commits, issues x actions. Tables over prose.
2. **Explicit assumptions** -- one-line bullets, prefixed `Assumption:`. Surface anything uncertain. If a karpathy-guidelines skill is loaded, name the section "Assumptions (per karpathy guidelines)".
3. **Success criteria** -- a checklist of verifiable outcomes (test counts, file paths, exit codes, gh issue states). Each item must be checkable by a single command.
4. **Skill acknowledgements** -- if a companion skill was loaded, briefly note how its principles were applied.
5. **Trailing question OR auto-proceed** -- if HARD gate: explicit yes/no or multi-choice question. If SOFT gate with open question: explicit question. If SOFT gate without open question: announcement that build will proceed.

### Build-phase behavior

Each build phase MUST:

1. **Re-read the plan** from conversation context (or summary if compressed; in Mode B, from the prompt the orchestrator threads in).
2. **Materialize the plan as a TodoWrite list** -- one todo per actionable item, plus a final verification todo.
3. **Execute sequentially**, updating todo statuses in real time.
4. **Loop on failure** -- if a command exits non-zero or a test fails, the build agent diagnoses, applies a targeted fix, and retries. It does NOT escalate back to plan unless the user issues a new intent.
5. **Emit a summary block at the end** -- what changed (file list), what passed (test counts / exit codes), what's the new state. The summary is the handoff signal to the next phase.

## Failure Policy

**Build phases loop internally.** They never escalate back to plan for the same scope.

When a command fails:
1. Read the failure output.
2. Diagnose the root cause (no need to ask the user).
3. Apply a targeted, surgical fix -- the minimum diff that resolves the failure.
4. Re-run the failed command (or the whole verification suite if the fix is broad).
5. Repeat until success criterion is met.

**Escalate back to plan only when:**
- The user issues a new intent that requires fresh investigation (not the same scope).
- The failure reveals that the plan was fundamentally wrong (rare; surface this explicitly to the user before re-planning).
- The build has looped 3+ times on the same failure without progress (signal to stop and ask).

## Companion Skills (Soft Dependencies)

Each phase has a recommended companion skill. The orchestrator loads the named skill at the start of the corresponding plan phase. If the companion is missing, fall back to the inline behavior described below.

| Phase | Companion skill              | Role                                  | Fallback if missing                                |
| ----- | ---------------------------- | ------------------------------------- | -------------------------------------------------- |
| 1     | karpathy-guidelines          | Shapes plan output (assumptions, surgical, criteria) | Inline the four karpathy principles (see below) |
| 1     | security-best-practices (opt)| Review if fixes touch auth/secrets    | Skip if no security-sensitive code in scope        |
| 2     | coding-agents-docs-guideline | Enforces doc style/structure          | Plain markdown edits, match existing doc style     |
| 3     | graphify                     | Runs knowledge graph regen (structural layer) | Skip sub-step 3a entirely                           |
| 3     | llm-wiki                     | Files durable insights as wiki pages (editorial layer) | Skip sub-step 3b entirely. See [references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md) |
| 4     | yeet                         | Standardizes commit + PR flow         | Inline git/gh commands following yeet's rules      |
| 5     | gh-address-comments          | Standardizes issue close/comment      | Inline `gh issue close` / `gh issue comment`       |

In Claude Code, load a companion skill at the start of its phase via the `Skill` tool (e.g. `Skill(yeet)` at PLAN-4). If the skill is not installed, fall back to the inline behavior.

### Inline karpathy principles (fallback if karpathy-guidelines skill is missing)

When the karpathy skill is unavailable, the plan-phase output for Phase 1 must still apply these four principles:

1. **Think before coding** -- surface assumptions explicitly. If uncertain, ask. If multiple interpretations exist, present them rather than picking silently.
2. **Simplicity first** -- minimum diff that solves the problem. No speculative abstractions, no flexibility that wasn't requested.
3. **Surgical changes** -- touch only what's necessary. Don't improve adjacent code, comments, or formatting. Match existing style.
4. **Goal-driven execution** -- every fix has a verifiable success criterion (a test, an exit code, a file content check). Loop until verified.

## Optional Skip Rules

Apply these checks at the start of each plan phase. If a skip condition is true, jump directly to the next non-skippable phase.

- **Phase 2 (docs)**: skip if `docs/` does not exist OR `ls docs/` returns empty.
- **Phase 3 (knowledge layer):**
  - Sub-step 3a (graphify): skip if `graphify-out/` does not exist OR the `graphify` skill is not installed.
  - Sub-step 3b (llm-wiki): skip if no wiki exists at `$WIKI_PATH` (default `~/wiki`) OR the `llm-wiki` skill is not installed. Also skip if Phase 1 produced no durable insights (see [references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md) for the checklist).
  - If both sub-steps are skipped, skip Phase 3 entirely.
- **Phase 4 (commit/PR)**: skip if not in a git repo (`git rev-parse --is-inside-work-tree` fails) OR no GitHub remote (`git remote get-url origin` fails). Report to user.
- **Phase 5 (issue triage)**: skip if `gh issue list --state open --limit 1` returns empty. Also skip if `gh auth status` fails and the user declines to authenticate.

If any phase is skipped, announce the skip with a one-line reason before moving on.

## Pre-Flight Checks

Before entering PLAN-1, the agent performs these one-time checks:

1. **Working directory** -- confirm the project root. Use `pwd` and `git rev-parse --show-toplevel`. If the user provided a path, cd there.
2. **Git status** -- `git status -sb`. Note any uncommitted changes (they will become part of the PR).
3. **GitHub CLI** -- `gh --version && gh auth status`. If unauthenticated, ask the user to run `gh auth login` before phase 4/5.
4. **Companion skills** -- quick discovery: which of the soft-dep skills are loadable? For Phase 3 specifically, probe both layers:
   - graphify: check for `graphify-out/` AND `graphify` skill loadable
   - llm-wiki: check for `$WIKI_PATH/SCHEMA.md` (default `~/wiki/SCHEMA.md`) AND `llm-wiki` skill loadable
   Cache all four booleans; this determines which Phase 3 sub-steps will run.
5. **Test runner discovery** -- find the project's test command (look for `package.json` scripts, `Makefile`, `tests/run.sh`, `pytest.ini`, etc.). Cache for BUILD-1 verification.

Report findings to the user as a one-block pre-flight summary, then enter PLAN-1.

## Worked Example

For a complete worked example (5 plan/build pairs, 11 issues addressed, 8 closed, 70/70 tests green, 6 commits, 1 PR), see [references/issue-resolution-workflow.md](references/issue-resolution-workflow.md).

## References

- [references/plan-build-primitives.md](references/plan-build-primitives.md) -- plan/build mechanics for delegated execution (Mode B): Task-tool subagents and `claude -p` headless, including advanced patterns (multi-round, parallel worktrees, plan-only, build-only)
- [references/issue-resolution-workflow.md](references/issue-resolution-workflow.md) -- The 6-phase FSM in detail, per-phase prompt shapes, failure/loop patterns, and a full worked example
- [references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md) -- Phase 3 in detail: graphify + llm-wiki division of labor, trigger conditions, handoff between structural and editorial layers
- [references/claude-code-cli-reference.md](references/claude-code-cli-reference.md) -- Complete `claude` CLI + subagent reference for delegated execution
