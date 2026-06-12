# The 6-Phase Issue Resolution Workflow

Deep dive on the lifecycle defined in [SKILL.md](../SKILL.md). Read this when you need:

- The full finite-state machine with state transitions and guard conditions
- Concrete prompt shapes for each phase
- Internal failure / loop patterns (what to do when tests fail mid-build)
- A worked example distilled from a real session

## 1. Finite-State Machine

```
START -> Pre-flight (pwd, git status, gh auth, skill + test-runner discovery)
 -> PLAN-1 Problem + Fix      -> HARD GATE (user: "yes" / "proceed" / "skip X")
 -> BUILD-1 Fix + Verify       (loop internally until verifications green)
 -> PLAN-2 Docs Gap            [SKIP if no docs/] -> SOFT GATE
 -> BUILD-2 Apply Docs
 -> PLAN-3 Knowledge Layer     [SKIP 3a if no graphify-out/ or skill; 3b if no wiki or skill]
 -> BUILD-3 3a graphify regen + 3b llm-wiki ingest
 -> PLAN-4 Commit Plan         [SKIP if no git/GH remote] -> SOFT GATE
 -> BUILD-4 Push + PR          (branch, N commits, push, gh pr create --draft)
 -> PLAN-5 Triage              [SKIP if no open issues] -> SOFT GATE
 -> BUILD-5 Close + Comment    (gh issue close x N, comment x M)
 -> END
```

### Transitions and guards

| From             | To                | Guard                                       | Trigger                                            |
| ---------------- | ----------------- | ------------------------------------------- | -------------------------------------------------- |
| START            | Pre-flight checks | (always)                                    | User invokes skill                                 |
| Pre-flight checks| PLAN-1            | Pre-flight summary reported                 | Auto                                               |
| PLAN-1      | BUILD-1     | User affirmative ("yes", "proceed")                       | HARD gate                                          |
| BUILD-1     | PLAN-2      | All Phase-1 success criteria met OR fallback ack         | Auto-chain                                         |
| PLAN-2      | BUILD-2     | No open question                                          | Auto (SOFT gate)                                   |
| PLAN-2      | (skip)      | `docs/` missing OR empty                                  | Skip-to-next                                       |
| BUILD-2     | PLAN-3      | Build summary emitted                                     | Auto-chain                                         |
| PLAN-3      | BUILD-3     | (always when reached)                                     | Auto                                               |
| PLAN-3      | (skip 3a)   | `graphify-out/` missing OR no graphify skill              | Skip sub-step 3a                                   |
| PLAN-3      | (skip 3b)   | No wiki at `$WIKI_PATH` OR no llm-wiki skill OR no durable insights | Skip sub-step 3b                        |
| PLAN-3      | (skip)      | Both 3a and 3b skipped                                    | Skip-to-next                                       |
| BUILD-3     | PLAN-4      | Build summary emitted                                     | Auto-chain                                         |
| PLAN-4      | BUILD-4     | No open question OR user answered commit-grouping        | Auto (SOFT gate)                                   |
| PLAN-4      | (skip)      | Not in git repo OR no GH remote                           | Skip-to-next                                       |
| BUILD-4     | PLAN-5      | Build summary emitted                                     | Auto-chain                                         |
| PLAN-5      | BUILD-5     | No open question OR user answered triage split            | Auto (SOFT gate)                                   |
| PLAN-5      | (skip)      | `gh issue list --state open` empty                        | Skip-to-END                                        |
| BUILD-5     | END         | All planned closes/comments applied                       | Auto                                               |

### Invariants (true for every phase pair)

1. **Single conversation context** -- the entire workflow runs in one Claude Code conversation. In Mode B the orchestrator stays in the main conversation and threads context into each subagent. No forks.
2. **Plan output ends with structured artifact + decision point** -- tables over prose, decisions over implicit assumptions.
3. **Build agent never escalates back to plan for the same scope** -- failures loop internally.
4. **Karpathy principles applied within each phase** -- assumptions in plan, surgical edits in build, verifiable criteria as todo goals.
5. **Each phase has at most one companion skill** bound to it.

---

## 2. Per-Phase Prompt Shapes

### 2.1 User prompt shape (when the user invokes the skill)

```
<imperative verb> <action> [on @artifact] [using <skill-name> skill]
```

Examples:
- *"address issues in https://github.com/owner/repo/issues using karpathy skill"*
- *"plan, fix, test, document, and PR the auth refactor"*
- *"ship the open issues release-ready"*

The orchestrator extracts:
- **Intent** -> drives which phases are needed
- **Artifact reference** -> `@docs`, `@graphify-out`, a URL, an issue tracker link
- **Skill invocation** -> trigger to load a companion skill for the relevant phase

### 2.2 Plan-phase output shape

Five mandatory sections (full schema in [SKILL.md](../SKILL.md#per-phase-template) and [agents/plan.md](../agents/plan.md)): `## Plan: <title>`, `### Findings` (table/matrix), `### Assumptions (per karpathy guidelines)`, `### Success Criteria` (single-command-checkable checklist), `### Skill Principles Applied`, `### Next Step` (gate question or "Proceeding to BUILD-N...").

### 2.3 Build-phase behavior shape

(1) Re-read plan from context; (2) TodoWrite goal list; (3) execute sequentially, updating statuses; (4) on failure: diagnose, surgical fix, retry — never escalate to plan; (5) emit the summary block:

```
## Build Summary
- Files changed: <list>
- Verifications: <test counts, exit codes, etc.>
- State: <what's true now that wasn't before>
```

### 2.4 Cross-phase handoff signal

The build-phase summary block is the signal that triggers the next phase. The orchestrator recognizes it and immediately transitions to plan mode for the next phase, announcing the transition.

---

## 3. Failure and Loop Patterns

### 3.1 Internal build loops (no plan re-entry)

When a verification fails during a build phase, the build agent enters an internal loop:

```
FAIL -> read failure output -> diagnose -> surgical fix -> retry -> (PASS | FAIL)
                                                                  |         |
                                                                  v         v
                                                              continue   loop back
```

**Concrete example** (from the reference session):

| Iteration | Test result                | Diagnosis                          | Fix applied                                     |
| --------- | -------------------------- | ---------------------------------- | ----------------------------------------------- |
| 1         | 62/70 passing (8 failures) | Read failing test logs in parallel | 8 surgical test edits (port-based checks, retry loops) |
| 2         | 66/70 passing (4 failures) | Inspected docker exec output       | env-var check for AC19; lenient assertions for VC7.x |
| 3         | 66/70 passing (still 4)    | `pgrep` returned wrong PID         | Created `get_gateway_pid()` helper in `common.bash`   |
| 4         | 70/70 passing              | --                                 | --                                              |

Three iterations of fix-and-retry, never escalating back to plan. The plan defined the success criterion (70/70 tests); the build looped until it was met.

### 3.2 When to escalate back to plan

Escalate only when:
1. **User issues a new intent** that requires fresh investigation (not the same scope).
2. **Plan was fundamentally wrong** -- rare. Surface this explicitly to the user before re-planning.
3. **Three or more iterations on the same failure without progress** -- stop and ask.

If the user issues a new intent mid-build, the build agent should:
- Complete or abort the current atomic operation cleanly (don't leave half-written files).
- Emit a final summary of what was completed.
- Acknowledge the new intent and transition to plan for the new scope.

### 3.3 Common failure categories and responses

| Failure type              | Example                                       | Response                                                |
| ------------------------- | --------------------------------------------- | ------------------------------------------------------- |
| Test failure              | `bash tests/run.sh` exits non-zero            | Read logs, surgical fix to test or code, retry           |
| Build failure             | `npm run build` exits non-zero                | Read compile errors, fix import/types, retry            |
| Command not found         | `gh: command not found`                       | Check `gh --version`; ask user to install if needed     |
| Auth failure              | `gh: authentication required`                 | Run `gh auth status`; ask user to authenticate          |
| Permission denied         | `git push` rejected                           | Check remote, branch protection; ask user               |
| Plan-fundamentally-wrong  | Fix doesn't address the real root cause       | Stop, surface explicitly, transition to plan            |

---

## 4. Compression Awareness

Long workflows may cross the context window limit; Claude Code handles this via auto-compact. (Mode B largely sidesteps it: each phase runs in a fresh subagent context and the orchestrator retains only the compact plan/Build-Summary artifacts.)

- **Boundaries:** compression ideally lands between phases (after a build summary, before the next plan), with the compression topic named for the phase it concludes.
- **Mid-phase:** if compression hits during a build, re-read the TodoWrite list (which survives compression) and the most recent build summary, then continue from the next incomplete todo.
- **Recovery anchors:** the build summary block carries enough state (files changed, criteria met) for the next phase to proceed even if the plan output was compressed away.

---

## 5. Worked Example

Distilled from a real session that took a repo with 11 open GitHub issues from "broken" to "all green, PR opened, 8 issues closed" in ~100 minutes.

### 5.1 Starting state

- Repo: containerized dev environment with docker-compose
- 11 open GitHub issues ranging from auth bugs to test flakiness
- Test suite: 70 E2E tests in `tests/e2e/*.bats`, currently passing 62/70

### 5.2 Phase execution log

| Phase | Wall-clock | Companion skill loaded    | Key output                                                                                  |
| ----- | ---------- | ------------------------- | ------------------------------------------------------------------------------------------- |
| PLAN-1 | ~4 min    | karpathy-guidelines       | 11-issue matrix with root cause + fix per issue. Assumptions: openssl in container, etc.    |
| (gate) | ~30 s     | --                        | User: "yes, also plan e2e tests + real-world test"                                          |
| BUILD-1| ~58 min   | security-best-practices   | 21 files edited, 4 test-loop iterations, 70/70 tests green. 4 compressions during this phase. |
| PLAN-2 | ~33 s     | karpathy + docs guideline | 7-doc gap matrix, surgical edit per doc                                                     |
| BUILD-2| ~2.5 min  | (same)                    | 7 docs surgically edited                                                                    |
| PLAN-3 | ~12 s     | graphify, llm-wiki       | Knowledge layer plan: graphify regen scope + 2 wiki pages queued (auth decision, settlement glossary) |
| BUILD-3| ~5 min    | (same)                    | graph.html/graph.json/GRAPH_REPORT.md regenerated; 2 wiki pages filed, 1 updated, index/log updated |
| PLAN-4 | ~50 s     | yeet                      | 6-commit breakdown table. Two open questions (commit grouping, include graphify)            |
| (gate) | ~5 s      | --                        | User: "yes all"                                                                             |
| BUILD-4| ~2 min    | (same)                    | Branch + 6 commits + push + `gh pr create --draft` -> PR #24                                |
| (idle) | ~18 min   | --                        | Human reviewed the PR                                                                      |
| PLAN-5 | ~7 s      | (none, uses gh)           | 11-issue triage table, recommended close 8 + comment 1 + leave 2 open                       |
| (gate) | ~5 s      | --                        | User: "follow your recommendation"                                                          |
| BUILD-5| ~47 s     | (none)                    | 8 issues closed with comments, 1 issue commented, 2 left open                               |

**Total wall-clock:** ~100 min (including 18 min human review idle).
**Total active agent time:** ~82 min.
**Tokens consumed:** ~780K input, ~75K output, ~78K reasoning.

### 5.3 Final outcomes

- 11 issues analyzed, 8 closed, 1 commented, 2 left open with explanatory comments
- 70/70 E2E tests green (up from 62/70)
- 21 files modified across code, tests, docs, config
- Branch + draft PR with 6 focused commits
- Knowledge layer updated (graph regenerated + 2 wiki pages filed)

### 5.4 What made it work

- **HARD gate after PLAN-1** caught a scope question (which upstream issues to defer)
- **SOFT gates** let phases 2-5 chain with minimal user input
- **Internal build loops** handled 3 rounds of test failures without user intervention
- **Compression** kept the workflow progressing through 4 context-window overflows
- **Companion skills** (karpathy, yeet, graphify, llm-wiki, docs guideline) standardized each phase's output

---

## 6. Subagent Fan-Out Rules

Some plan phases benefit from parallel reads. Use Claude Code's **Task tool** to fan out read-only subagents (launch several Task calls in a single turn to run them concurrently).

### 6.1 When to fan out

- **Plan-phase reads can fan out** -- e.g. reading all docs in parallel for docs-gap analysis
- **Build-phase writes always stay in the main agent** -- serial execution prevents file conflicts

### 6.2 What to fan out

| Phase   | Read-heavy subtasks                                  | Subagent pattern                            |
| ------- | ---------------------------------------------------- | ------------------------------------------- |
| PLAN-1  | Explore repo structure, find affected files          | One `Explore` subagent per concern          |
| PLAN-2  | Read all docs in parallel                            | One `Explore` subagent per 5-7 docs         |
| PLAN-3  | Semantic extraction for graph nodes                  | Multiple `Explore` subagents, chunked       |
| PLAN-4  | Read staged diff, classify changes per commit        | Single subagent for diff classification     |
| PLAN-5  | Read all open issues in parallel                     | One subagent fetches, main agent classifies |

### 6.3 What NEVER to fan out

- File edits (build phase)
- Git commands (commit, push)
- `gh issue close` / `gh issue comment` -- these have side effects and ordering matters
- Any command with side effects on the world outside the local repo

---

## 7. Cross-References

- [SKILL.md](../SKILL.md) -- Main skill, the 6-phase lifecycle and approval gates
- [plan-build-primitives.md](plan-build-primitives.md) -- plan/build mechanics for delegated execution mode (Task tool + `claude -p`)
- [claude-code-cli-reference.md](claude-code-cli-reference.md) -- Complete `claude` CLI + subagent flag reference
