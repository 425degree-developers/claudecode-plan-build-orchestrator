# Knowledge Layer Workflow: graphify + llm-wiki

Phase 3 of the orchestrator builds and maintains the project's **knowledge layer** --
the persistent record of what the codebase is (structure) and what we have learned
about it (synthesis). Two companion skills cover non-overlapping concerns:

| Sub-step | Skill      | Question it answers                        | Output                          | Regenerable from source? |
| -------- | ---------- | ------------------------------------------ | ------------------------------- | ------------------------ |
| 3a       | graphify   | "What is structurally in this codebase?"   | `graphify-out/` graph + report  | Yes (AST + LLM extract)  |
| 3b       | llm-wiki   | "What have we learned that isn't in code?" | Pages in `$WIKI_PATH`           | No (editorial only)      |

Both are **soft dependencies**. Phase 3 runs whichever sub-steps have their
prerequisites met; it skips the rest.

## 1. Division of Labor

### graphify -- the structural layer

graphify turns a folder of code, docs, papers, images, or video into a queryable
knowledge graph with community detection and an honest audit trail (every edge
tagged EXTRACTED / INFERRED / AMBIGUOUS with a confidence score). It is the
**map** of what exists.

Use it for:
- Structural queries: "what depends on X", "shortest path from Auth to DB"
- Cross-document surprise: community detection finds links across modules
  that you would not think to ask about directly
- Agent-facing surface: the MCP server lets other agents query the graph
  without re-reading source files
- Cheap regen: `--update` re-extracts only changed files; code-only changes
  skip the LLM entirely (AST-only)

graphify output regenerates from source on every run. Anything you "know" about
the codebase that is reconstructable from the files themselves belongs in the
graph, not in the wiki.

### llm-wiki -- the editorial layer

llm-wiki is a curated markdown knowledge base (compatible with Obsidian) that
compounds editorial decisions over time. It is the **journal** of what you have
learned that source files cannot tell you.

Use it for:
- **Decision rationale** -- why approach X was chosen over Y, what was rejected
- **Discovered constraints** -- "Module A silently depends on Module B's init
  order" (not visible in the graph)
- **Comparisons** -- "AuthModule and GatewayAuth solve the same problem; use
  AuthModule for internal services, GatewayAuth for federated"
- **Postmortems** -- "we tried X, it failed because Y, we now do Z"
- **Glossary** -- domain terms that appear in code but mean something specific
  in this project

llm-wiki content does NOT regenerate from source. Every page is an editorial
act. If you do not write it down, you will re-derive it (or fail to) next time.

## 2. When to Use Each

```
                  Is the knowledge reconstructable
                  from reading the source files?
                           |
              +------------+------------+
              | YES                     | NO
              v                         v
        graphify covers it         llm-wiki covers it
        (or neither, if the        (or neither, if it
        graph is not maintained)   is not worth filing)
```

### Concrete examples for a large codebase

| Situation                                                   | graphify | llm-wiki |
| ----------------------------------------------------------- | -------- | -------- |
| "What imports `auth.py`?"                                   | Yes      | No       |
| "Shortest dependency path from `UserAPI` to `Database`"     | Yes      | No       |
| "Which community does `PaymentProcessor` cluster with?"     | Yes      | No       |
| "Why does `auth.py` check `request.headers['X-User-Id']`?"  | No       | Yes      |
| "Why did we reject JWT in favor of session cookies?"        | No       | Yes      |
| "What did we learn from the May 2026 auth outage?"          | No       | Yes      |
| "`PaymentProcessor` and `RefundHandler` overlap -- when to use each?" | Partial (overlap visible) | Yes (decision rule) |
| "What does 'settlement' mean in this codebase?"             | No       | Yes      |
| "List every module that calls into the legacy `billing/` tree" | Yes   | No       |

## 3. Trigger Conditions (within Phase 3)

### Sub-step 3a: graphify regen

**Run when ALL of:**
- `graphify-out/` exists in the project root, OR the user has invoked graphify
  on this project before
- The `graphify` skill is installed and loadable
- Phase 1 (fix+verify) modified any file tracked by graphify (code, docs, configs)

**Skip when ANY of:**
- `graphify-out/` does not exist AND the user has not asked to initialize it
- The `graphify` skill is missing (fall back: skip entirely, do not try to
  build a graph manually)

### Sub-step 3b: llm-wiki ingest

**Run when ALL of:**
- A wiki exists at `$WIKI_PATH` (default `~/wiki`) -- check for `SCHEMA.md`
  and `index.md` at that path
- The `llm-wiki` skill is installed and loadable
- Phase 1 produced at least one **durable insight** (see checklist below)

**Skip when ANY of:**
- No wiki exists (do not initialize one mid-run without explicit user consent)
- The `llm-wiki` skill is missing
- Phase 1 was routine (no non-obvious decisions, no rejected alternatives, no
  discovered constraints worth filing)

#### Durable insight checklist

Before filing anything, ask whether Phase 1 produced any of:

1. **Architectural decision with a rejected alternative** -- "we chose X over Y
   because Z". If PLAN-1 surfaced assumptions that became the basis for the
   chosen fix, those assumptions are candidates for a wiki page.
2. **Discovered constraint not visible in code** -- "this module must init
   before that one because of a shared in-memory state". If the constraint
   would surprise someone reading the code cold, file it.
3. **Workaround for an upstream issue** -- "we pin libfoo==1.2.3 because 1.3
   breaks our use case (see upstream bug #N)".
4. **Domain-specific term that code uses in a non-standard way** -- "in this
   codebase, 'settlement' means X, not the financial-industry standard Y".
5. **Cross-module comparison that the graph surfaces as a surprising link** --
   if graphify's "Surprising Connections" section flags two modules that
   cluster together unexpectedly, and the connection reflects a real
   architectural pattern, file a comparison page.

If none of these apply, skip the wiki ingest. Routine fixes (typo, test
flakiness, dependency bump) do not produce durable insight.

## 4. The Handoff: graph -> wiki

graphify and llm-wiki are not just parallel -- they feed each other.

### graphify seeds wiki topics

After sub-step 3a, read `graphify-out/GRAPH_REPORT.md`. Three sections are
direct candidates for wiki ingest:

1. **God Nodes** -- high-degree nodes often correspond to architectural
   concepts worth a wiki page. If `auth_middleware` is a god node, file a
   `concepts/auth-architecture.md` page that explains *why* it is central
   (graphify tells you it is central; the wiki tells you why).
2. **Surprising Connections** -- cross-community edges that the agent would not
   have thought to ask about. Each surprising connection is a candidate for a
   `comparisons/` page explaining the link.
3. **Low-cohesion communities** -- communities with low cohesion scores often
   indicate code that was grouped by accident but should be re-examined. Worth
   a `concepts/` page diagnosing the smell.

### wiki cites graph nodes

When writing a wiki page, cite graph nodes for the structural claims:

```markdown
The auth middleware ([[node:auth-middleware]]) is the single highest-degree
node in the codebase (graphify god node, degree=47). It gates every
authenticated route and shares session state with the user service
([[node:user-service]]). We chose session cookies over JWTs in 2026-04
because [...]
```

This lets a future reader jump from the wiki's *why* to the graph's *what* in
one click.

## 5. Execution Sequence

Phase 3 runs sub-steps 3a and 3b **sequentially**, not in parallel:

```
PLAN-3
  |
  +-- Plan graph regen scope (which files changed? --update?)
  +-- Plan wiki ingest scope (any durable insights from Phase 1?)
  |
BUILD-3
  |
  +-- 3a: graphify --update (or full /graphify if first run on this repo)
  |       - output: graphify-out/graph.json, graph.html, GRAPH_REPORT.md
  |
  +-- 3b: llm-wiki ingest (if durable insights exist AND wiki exists)
          - read GRAPH_REPORT.md -- check god nodes, surprising connections
          - for each durable insight from Phase 1, file or update a page
          - update SCHEMA.md (if new tags needed) and index.md
          - append to log.md
```

Sequential because sub-step 3b often reads the output of 3a (specifically
`GRAPH_REPORT.md`) to decide whether graphify surfaced something worth a wiki
page.

## 6. What Not to Do

- **Do not** use llm-wiki as a codebase map. It has no AST, no graph traversal,
  and ingesting every Python file as an entity page is malpractice. Use
  graphify.
- **Do not** write editorial content into `graphify-out/`. The next `--update`
  will clobber it. Editorial content goes in the wiki.
- **Do not** file a wiki page for every fix. Routine work is not durable
  insight. When in doubt, skip -- you can always file later.
- **Do not** initialize a wiki mid-run without explicit user consent. The
  wiki is a long-term commitment; it should be set up deliberately, not as
  a side effect of a fix-and-PR run.
- **Do not** re-derive structural knowledge in wiki pages. "Module A imports
  Module B" belongs in the graph (where it auto-updates), not in a wiki page
  (where it will go stale).

## 7. Companion Skill Discovery

During pre-flight checks (before Phase 1), the orchestrator should probe for
both skills:

```
graphify:   check for graphify-out/ AND graphify skill loadable
llm-wiki:   check for $WIKI_PATH/SCHEMA.md AND llm-wiki skill loadable
```

Cache the four booleans. They determine which sub-steps of Phase 3 will run
and which will be skipped.

Report the discovery result in the pre-flight summary:

```
Knowledge layer:
  graphify: ready (graphify-out/ exists, skill loaded)
  llm-wiki: ready (wiki at ~/wiki, skill loaded)
```

or:

```
Knowledge layer:
  graphify: skip (no graphify-out/)
  llm-wiki: skip (no wiki at ~/wiki)
```

## 8. Reference

- graphify skill: https://github.com/safishamsi/graphify
- llm-wiki skill: based on Karpathy's LLM Wiki pattern
  (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- In Claude Code, both skills are loaded via the `Skill` tool
  (`Skill(graphify)` and `Skill(llm-wiki)`) when they are installed.
