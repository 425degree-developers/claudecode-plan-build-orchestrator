# Plan / Build Primitives (Claude Code)

Mechanics for orchestrating the `plan` and `build` agents inside Claude Code. This is the **delegated execution mode** reference (Mode B). For the higher-level 6-phase lifecycle (problem -> fix -> docs -> ship -> close), see [SKILL.md](../SKILL.md).

Claude Code gives you two ways to delegate a phase:

1. **In-process subagents via the Task tool** (recommended) — the bundled `plan` and `build` agents run in isolated context windows within the same Claude Code session.
2. **Cross-process headless via `claude -p`** — shell out to a fresh Claude Code process per phase, resuming a session id. This is the literal analog of opencode's `opencode run -s`.

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

## Headless mode: `claude -p --resume`

For cross-process delegation (orchestrator is an external script, or you want each phase in its own OS process):

```bash
# Phase 1: Plan — run in plan permission mode, capture the session id
claude -p "Plan: <feature>. Produce the Phase-1 plan artifact." \
  --permission-mode plan --output-format json > plan1.json
SESSION_ID=$(jq -r '.session_id' plan1.json)

# Phase 1: Build — resume the SAME session so plan context carries over
claude -p --resume "$SESSION_ID" "Implement the approved plan. Loop until tests pass."

# Phase 2: Plan — still the same session
claude -p --resume "$SESSION_ID" "Check docs gaps and plan surgical updates."
```

`--resume <id>` is the analog of `opencode run -s <id>`. `--continue` resumes the most recent session (analog of `opencode run -c`). See [claude-code-cli-reference.md](claude-code-cli-reference.md) for the full flag list.

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
