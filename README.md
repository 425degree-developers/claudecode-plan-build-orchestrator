# claudecode-plan-build-orchestrator

An [Agent Skills](https://agentskills.io/) compatible skill that drives a **6-phase lifecycle** (problem -> fix + verify -> docs sync -> knowledge graph -> commit + PR -> issue triage) by alternating plan and build modes. Built for **Claude Code**: the plan/build separation maps onto Claude Code's native **Plan Mode** and **subagents**, taking a batch of issues or a feature request all the way from "broken" to "shipped" in one autonomous run.

> Claude Code port of [bachkukkik/opencode-plan-build-orchestrator](https://github.com/bachkukkik/opencode-plan-build-orchestrator). Same 6-phase lifecycle, gates, failure policy, and companion skills; the delegation mechanics are rebuilt around Claude Code's Task-tool subagents and `claude -p` headless mode instead of `opencode run`.

## What It Does (v1.0.0)

The skill defines a **deterministic pipeline of plan/build pairs**:

| # | Phase           | Plan output                          | Build artifacts                                |
| - | --------------- | ------------------------------------ | ---------------------------------------------- |
| 1 | Problem + Fix   | Root-cause matrix, assumptions, criteria | Code/test/config edits; verifications green |
| 2 | Docs Gap        | Docs x gap matrix with priority      | Surgical edits to docs                         |
| 3 | Knowledge Layer | Knowledge layer update plan (graph + wiki) | Regenerated `graphify-out/` + filed wiki pages for durable insights |
| 4 | Commit + PR     | N-commit breakdown, file -> issue mapping | Branch + commits + push + draft PR         |
| 5 | Issue Triage    | Status table per issue               | `gh issue close`/`comment`                     |

**Approval gates:** one HARD gate after PLAN-1 (user must approve scope); SOFT gates thereafter (phases auto-chain unless the agent has a literal open question).

**Failure policy:** build phases loop internally on test/command failures. They never escalate back to plan unless the user issues a new intent or 3+ retries fail.

## Two Execution Modes

- **Mode A -- Harness-native (default, recommended):** the main agent alternates thinking/acting per phase. Plan phases run in Claude Code **Plan Mode** (read-only, ends with the `ExitPlanMode` approval prompt — a natural fit for the PLAN-1 HARD gate); build phases run with the full tool set. No subagents, no shelling out.
- **Mode B -- Subagent-delegated (optional):** each phase runs in an isolated context via the **Task tool**, dispatching the bundled `plan` (read-only) and `build` (read+write+execute) subagents. The main conversation threads context between phases. A `claude -p --resume` headless variant is documented for cross-process delegation.

Mode A is described inline in [SKILL.md](SKILL.md). Mode B is documented in [references/plan-build-primitives.md](references/plan-build-primitives.md) and [references/claude-code-cli-reference.md](references/claude-code-cli-reference.md).

## Soft Dependencies (Companion Skills)

Each phase has an optional companion skill. The skill gracefully falls back to inline behavior when a companion is missing.

| Phase | Companion skill              |
| ----- | ---------------------------- |
| 1     | `karpathy-guidelines`        |
| 1 (opt) | `security-best-practices`  |
| 2     | `coding-agents-docs-guideline` |
| 3     | `graphify` (structural layer)   |
| 3     | `llm-wiki` (editorial layer)    |
| 4     | `yeet`                       |
| 5     | `gh-address-comments`        |

## Quick Start

### Install

Copy the `claudecode-plan-build-orchestrator/` directory into your Claude Code skills folder:

- Project scope: `.claude/skills/claudecode-plan-build-orchestrator/`
- User scope: `~/.claude/skills/claudecode-plan-build-orchestrator/`

```bash
git clone https://github.com/425degree-developers/claudecode-plan-build-orchestrator
mkdir -p .claude/skills
cp -r claudecode-plan-build-orchestrator .claude/skills/
```

### Install the subagents (for Mode B)

To use the delegated execution mode, also copy the bundled subagents into your agents folder:

```bash
mkdir -p .claude/agents
cp .claude/skills/claudecode-plan-build-orchestrator/agents/*.md .claude/agents/
```

This registers the `plan` (read-only) and `build` (read+write+execute) subagents so the orchestrator can dispatch them via the Task tool. Mode A needs no agent install.

> **Restart Claude Code after copying the agents.** Newly added `.claude/agents/*.md` files are only picked up at startup — until you restart, `subagent_type: "plan"`/`"build"` won't resolve (and `"plan"` may silently fall back to the built-in `Plan` agent).

### Use It

Tell Claude Code any of:

- "Address the open issues in this repo"
- "Plan, fix, test, document, and PR the auth refactor"
- "Ship it release-ready"

Claude loads this skill and runs the 6-phase pipeline. You approve once after the initial plan; everything else chains autonomously.

## Directory Structure

```
claudecode-plan-build-orchestrator/
├── SKILL.md                                  # Main skill (the 6-phase lifecycle)
├── LICENSE                                   # Apache 2.0
├── README.md                                 # This file
├── agents/
│   ├── plan.md                               # Read-only plan subagent (Mode B)
│   └── build.md                              # Read+write+execute build subagent (Mode B)
└── references/
    ├── plan-build-primitives.md              # Plan/build mechanics for Mode B (Task tool + claude -p)
    ├── issue-resolution-workflow.md          # FSM deep dive + worked example
    ├── knowledge-layer-workflow.md           # Phase 3 graphify + llm-wiki division of labor
    └── claude-code-cli-reference.md          # Complete claude CLI + subagent reference
```

## Example: End-to-End Run

```
You: "Address all open issues in this repo using karpathy skill"

Claude:
  Pre-flight: pwd, git status, gh auth, skill discovery, test runner discovery
  PLAN-1 (Plan Mode): 11-issue root cause matrix + assumptions + criteria
  -> HARD GATE (ExitPlanMode)
You: "yes"
  BUILD-1: 21 files edited, 4 internal loops, 70/70 tests green
  PLAN-2: 7-doc gap matrix (auto-chain)
  BUILD-2: 7 surgical doc edits
  PLAN-3: knowledge layer plan (graphify regen + llm-wiki ingest) (auto-chain)
  BUILD-3: graph regenerated + 2 wiki pages filed (auth architecture, settlement glossary)
  PLAN-4: 6-commit breakdown -> "yes all"
  BUILD-4: branch + 6 commits + push + draft PR #24
  PLAN-5: triage table -> "follow recommendation"
  BUILD-5: 8 issues closed, 1 commented, 2 left open with reason
  END
```

For the full worked example with timing, token usage, and failure-loop traces, see [references/issue-resolution-workflow.md](references/issue-resolution-workflow.md).

## Phase 3: Knowledge Layer (graphify + llm-wiki)

Phase 3 maintains two non-overlapping layers of project knowledge. Both are
optional and run independently.

| Sub-step | Skill    | Layer     | Captures                                       | Regenerable? |
| -------- | -------- | --------- | ---------------------------------------------- | ------------ |
| 3a       | graphify | Structural | Imports, calls, communities, god nodes         | Yes (from source) |
| 3b       | llm-wiki | Editorial  | Decision rationale, comparisons, glossary, postmortems | No (compounds over time) |

**Rule of thumb:** if the knowledge is reconstructable from reading the code,
it belongs in graphify. If it is not, it belongs in the wiki. If neither skill
is available, Phase 3 is skipped.

See [references/knowledge-layer-workflow.md](references/knowledge-layer-workflow.md)
for the complete division of labor, trigger conditions, and the handoff between
the two layers.

## What Changed vs. the opencode Original

This is a faithful Claude Code port. The 6 phases, ASCII pipeline, gates, failure policy, skip rules, pre-flight checks, and companion-skill list are unchanged. The differences are all in delegation mechanics:

- **Mode A** now leans on Claude Code's real **Plan Mode** for plan phases (read-only enforced at the harness level; `ExitPlanMode` doubles as the PLAN-1 HARD gate), instead of opencode's manual TUI plan/build toggle.
- **Mode B** dispatches dedicated **subagents via the Task tool** (`agents/plan.md`, `agents/build.md`) instead of `opencode run --agent plan|build`. Because Task subagents don't share a session, the orchestrator threads context between phases through the prompt rather than via opencode's `-s <session_id>`.
- A **`claude -p --resume` headless** variant replaces `opencode run -s` for cross-process delegation.
- `references/claude-code-cli-reference.md` replaces `opencode-cli-reference.md`.
- License changed to **Apache 2.0**.

## Specification

This skill follows the [Agent Skills specification](https://agentskills.io/specification):

- `SKILL.md` with valid YAML frontmatter (`name`, `description`, `license`, `compatibility`, `metadata`)
- `name` matches directory name: `claudecode-plan-build-orchestrator`
- `description` under 1024 characters
- Progressive disclosure: main instructions in SKILL.md, details in references/

## Credits

Ported from [bachkukkik/opencode-plan-build-orchestrator](https://github.com/bachkukkik/opencode-plan-build-orchestrator) (MIT). This Claude Code port is maintained by [425degree-developers](https://github.com/425degree-developers).

## License

Apache 2.0 — see [LICENSE](LICENSE).
