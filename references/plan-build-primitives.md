# Plan / Build Primitives (Claude Code)

Mechanics for orchestrating the `plan` and `build` agents inside Claude Code. This is the **delegated execution mode** reference (Mode B). For the higher-level 6-phase lifecycle (problem -> fix -> docs -> ship -> close), see [SKILL.md](../SKILL.md).

Claude Code gives you two ways to delegate a phase — pick by who the orchestrator is:

1. **In-process subagents via the Task tool** (when Claude Code itself orchestrates) — the bundled `plan` and `build` agents run in isolated context windows within the same Claude Code session. No process spawn, no re-auth, no config reload.
2. **Cross-process headless via `claude -p`** (when an external agent like Hermes orchestrates) — shell out to a fresh Claude Code process per phase, resuming a session id. This is the literal analog of opencode's `opencode run -s`. The Task tool does not exist outside a Claude Code session, so this is the only delegation path for external orchestrators — and each call must use the lean-worker flags below.

## The agents

| Agent    | Role                                              | Writes Code? | Tools                                            |
| -------- | ------------------------------------------------- | ------------ | ------------------------------------------------ |
| `plan`   | Reads codebase, designs approach, asks questions  | No           | Read, Glob, Grep, WebFetch, read-only Bash       |
| `build`  | Implements plans, writes files, runs tests        | Yes          | Read, Glob, Grep, Edit, Write, Bash, TodoWrite   |
| `Explore`| Lightweight codebase exploration (built-in)       | No           | Read-only search tools                           |

The `plan` and `build` agents ship with this skill in [../agents/](../agents/). Install them to `.claude/agents/` (project) or `~/.claude/agents/` (user) so the Task tool can dispatch them by name.

## Core flow (one plan/build pair)

### Phase A: Plan

Dispatch the read-only plan agent:

```
Task(subagent_type="plan",
     prompt="Investigate <area> and design a plan to <goal>. Produce the Phase-N plan artifact.")
```

It returns a structured plan artifact (findings table, assumptions, success criteria) as its final message. Because subagents do not share the main context, **the artifact it returns is the only thing that survives** — the orchestrator keeps it in the main conversation.

### Phase B: Review

The orchestrator (the main agent) reviews the plan output:
- Is the design sound?
- Are there open questions to resolve?
- Does the user need to approve? (HARD gate after PLAN-1.)

### Phase C: Iterate (optional)

Dispatch another plan round. Since subagents are stateless, **include the prior plan in the new prompt**:

```
Task(subagent_type="plan",
     prompt="Here is the current plan:\n<plan>\nRefine it to also handle <new concern>.")
```

### Phase D: Build

Dispatch the build agent with the approved plan threaded in:

```
Task(subagent_type="build",
     prompt="Implement this approved plan:\n<plan>\nLoop until <success criteria> are green. Emit a Build Summary.")
```

### Phase E: Verify

The orchestrator verifies the build output:
- Check files created/modified (the Build Summary lists them).
- Run tests if applicable.
- Confirm the original requirements are met.

## Context threading (the key difference from opencode)

opencode preserved plan context across phases with `-s <session_id>`. Claude Code Task subagents **start fresh every call** — there is no shared session. So:

> The orchestrator (main conversation) is the single source of truth. Each phase's subagent returns a self-contained artifact (plan or Build Summary); the orchestrator stores it and threads the relevant pieces into the next subagent's prompt.

Practically:
- A plan subagent's full plan artifact -> goes into the build subagent's prompt.
- A build subagent's Build Summary -> goes into the next plan subagent's prompt.

If you want true session persistence across processes, use the `claude -p --resume` headless mode below instead of the Task tool.

## Headless mode: `claude -p --resume` (Mode C — external orchestrators like Hermes)

For cross-process delegation (orchestrator is an external agent or script), always use the **lean worker** flag set — a bare `claude -p` inherits the user's interactive model pin, all plugins/MCP servers, and `ask` permission rules that headless mode silently denies:

```bash
LEAN="--setting-sources project --strict-mcp-config \
      --output-format stream-json --verbose --max-budget-usd 5"

# Phase 1: Plan — STRONG model (requirements + root-cause diagnosis); capture session id
claude -p "Plan: <feature>. Produce the Phase-1 plan artifact." \
  --permission-mode plan --model opus $LEAN > plan1.ndjson 2>&1
SESSION_ID=$(jq -r 'select(.type=="system" and .subtype=="init") | .session_id' plan1.ndjson | head -1)

# Phase 1: Build — GENERAL model (executes the approved plan); re-pass flags every call
claude -p --resume "$SESSION_ID" "Implement the approved plan. Loop until tests pass." \
  --permission-mode bypassPermissions --model sonnet $LEAN > build1.ndjson 2>&1 &

# Phase 2: Plan — GENERAL model (gap matrix derived from Phase-1 output)
claude -p --resume "$SESSION_ID" "Check docs gaps and plan surgical updates." \
  --permission-mode plan --model sonnet $LEAN > plan2.ndjson 2>&1

# Phase 4: Commit + PR — WEAKEST model (pure git/gh mechanics)
claude -p --resume "$SESSION_ID" "Execute the commit plan: branch, commits, push, draft PR." \
  --permission-mode bypassPermissions --model haiku $LEAN > build4.ndjson 2>&1

# Phase 5: Triage plan — STRONG again (diagnosis against the codebase); BUILD-5 close/comment -> haiku
```

Model per phase follows the [Model Tiers table in SKILL.md](../SKILL.md#model-tiers-per-phase): `opus` for PLAN-1/PLAN-5, `sonnet` for builds and derived plans, `haiku` for commit/PR/close mechanics.

Rules of thumb for the orchestrator:

- **Run build workers in the background** and poll the NDJSON — build phases legitimately take minutes; don't block a foreground pipe.
- **One concern per call** — split "waves" of unrelated tasks into separate worker calls so failures are isolated and progress is observable.
- **Large artifacts via scripts** — never instruct a worker to hand-type a big fixture/dataset; have it write and run a generator script.
- **Re-pass flags on every `--resume`** — the session restores conversation context, not configuration.

`--resume <id>` is the analog of `opencode run -s <id>`. `--continue` resumes the most recent session (analog of `opencode run -c`). See [claude-code-cli-reference.md](claude-code-cli-reference.md) for the full flag list and the lean-worker rationale.

## Decision framework

### When to plan first

- New feature in existing codebase -> **plan -> build**
- Bug with unknown root cause -> **plan -> build**
- Architecture change or refactor -> **plan (2-3 rounds) -> build**
- User asks "how should we..." -> **plan only**

### When to build directly

- Simple, well-defined task -> **build only**
- Quick script or one-off -> **build only**
- User says "just do it" -> **build only**

### How many plan rounds

| Complexity    | Rounds | Example                                    |
| ------------- | ------ | ------------------------------------------ |
| Simple        | 1      | Add a CLI flag to existing tool            |
| Medium        | 1-2    | New API endpoint with tests                |
| Complex       | 2-3    | Database migration + schema change         |
| Architectural | 3+     | Refactor module boundaries, change build system |

## Advanced patterns

### Multi-round planning (complex features)

Thread the evolving plan through each round (subagents are stateless):

```
Task(plan,  "Investigate the auth module and design a plan to add OAuth2 support.")
  -> returns plan v1
Task(plan,  "Plan v1:\n<plan v1>\nAlso handle token refresh; add the refresh flow.")
  -> returns plan v2
Task(plan,  "Plan v2:\n<plan v2>\nFinalize: add error handling, migration path, rollback.")
  -> returns final plan
Task(build, "Implement this plan:\n<final plan>\nInclude tests for OAuth2 + token refresh.")
```

### Parallel features with git worktrees

For independent features that can be developed simultaneously, isolate each in its own git worktree so builds don't conflict:

```bash
git worktree add ../feature-a -b feature-a
git worktree add ../feature-b -b feature-b
```

Then run each plan/build pair against its own worktree directory. In headless mode, pass the worktree path:

```bash
claude -p --resume "$SESSION_A" "Implement plan." --add-dir ../feature-a
claude -p --resume "$SESSION_B" "Implement plan." --add-dir ../feature-b
```

**Never share one workdir across parallel builds.** Use worktrees for isolation within the same repo.

### Plan -> build -> plan (iterative development)

When a build reveals an issue that needs re-planning, capture the Build Summary and feed it into a new plan round:

```
Task(plan,  "Plan migration from Flask to FastAPI.")              -> plan
Task(build, "Implement this plan:\n<plan>")                       -> Build Summary reveals
                                                                     session middleware incompatibility
Task(plan,  "Build summary:\n<summary>\nThe session middleware
             isn't FastAPI-compatible. Design a fix.")            -> updated plan
Task(build, "Implement the updated plan:\n<updated plan>")
```

### Plan-only (investigation)

When you just need understanding, not code changes — dispatch only the plan agent and report its findings to the user. No build phase.

```
Task(plan, "Investigate why /api/search is slow. Look at queries, indexing,
            caching. Report findings with file:line references and recommendations.")
```

### Build-only (simple task)

When planning isn't needed, skip straight to build:

```
Task(build, "Fix the typo in README.md: 'recieve' -> 'receive'.")
```

## Orchestration loop (main agent)

```
1. Receive user prompt
2. Assess complexity -> decide: plan+build or build-only
3. If plan+build:
   a. Task(plan, "<prompt>") -> capture plan artifact in main context
   b. Review plan quality:
      - Missing edge cases? -> another plan round with feedback threaded in
      - Open questions? -> ask user (HARD gate after PLAN-1) or resolve autonomously
      - Good to go? -> proceed to build
   c. Task(build, "Implement this plan:\n<plan>") -> capture Build Summary
   d. Verify output (check files, run tests)
   e. Report to user
4. If build-only:
   a. Task(build, "<prompt>")
   b. Verify output
   c. Report to user
```

## Pitfalls

- **Subagents don't share context** — always thread the prior plan / Build Summary into the next subagent's prompt. This is the #1 difference from opencode's `-s` sessions.
- **Plan agent can't write** — its tools exclude Edit/Write and its Bash is read-only. Send it design work, not implementation.
- **Build agent needs the plan** — it has no memory of the plan phase; the orchestrator must hand it the full plan in the prompt.
- **In headless mode, capture `session_id`** — without `--resume <id>` (or `--continue`), each `claude -p` is a brand-new session and context is lost.
- **Workdir / worktree matters** — never run parallel builds in the same directory.

## Verification checklist

After build completes:

1. Confirm the Build Summary lists the files it claims to have changed (`git status`).
2. Verify files created/modified in the workdir.
3. Run tests if applicable.
4. Review key changes for correctness.
5. Report concrete outcomes to the user.
