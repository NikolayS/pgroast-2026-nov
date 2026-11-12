# Design pre-review: Native planner and executor for SQL/PGQ

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7258](https://commitfest.postgresql.org/patch/7258/); [thread](https://www.postgresql.org/message-id/flat/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal of running VLE patterns is reasonable, and no surviving objection is fatal. The design leaves open questions that the list must settle before code review. The native path is on by default while message #1 says row level security was not started (security-1). The series keeps two execution paths behind a GUC (maintenance-fit-1) and moves fixed-length patterns off the join planner without numbers (demand-1).

Not checked: Without the code, RLS enforcement in the inner subplans, deparsing of VLE quantifiers, and the context and upper bound of the depth GUC could not be checked. Nine of the 12 thread messages were seen only through the objections.

## Prior attempts

None found before 2026-08-08.

## Objections a reviewer is likely to raise

### 1. Native path on by default without row level security enforcement

A DBA puts RLS policies on an edge or vertex table and grants SELECT on a property graph over it. A restricted user runs GRAPH_TABLE with a VLE pattern; the native GraphScan path, enabled by default, may return rows the policy hides. Nothing reports an error, so the leak goes unnoticed until someone audits the results.

Message #1 says row level security was not approached at all. Messages #2 to #12 add partitioning, views, foreign tables and lateral handling but never say RLS is enforced. The rewriter gets RLS and per-table permission checks from ordinary relational range table entries. GraphScan builds its own inner subplans per element table, and the proposal does not say those subplans still carry the RLS quals and permission checks while enable_native_graphtable defaults to on.

**Question:** Does the native path apply the same RLS policies and table privilege checks as the rewriter, with tests that show identical rows for a restricted role?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJyuutgJzqVJEsKGFaQpPbp812TcAx69wpgSuHjn6vVDmEdBVw%40mail.gmail.com); `security-1`</sub>

### 2. Two graph execution paths with a GUC that switches between them

A later patch fixes a GRAPH_TABLE semantics bug or adds a pattern feature in rewriteGraphTable.c and does not touch nodeGraphScan.c. Results now depend on enable_native_graphtable. A user finds out when the two settings return different rows for the same query.

Message #2 says the series aims to replace the rewriter completely, yet enable_native_graphtable keeps the rewriter as a live, user-selectable path. The two paths already diverged once: v2 had to copy the rewriter's lateral results instead of the documented Cartesian product (messages #1, #2). Nothing makes a change to one path fail when the other is not updated. The proposal does not say when the rewriter path goes away, or how the two are kept in agreement until then.

**Question:** Is the rewriter path removed in this series, and if not, what keeps both paths returning the same results until it is?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Per-depth copies of arbitrary subplans break the one-PlanState-per-Plan assumption

Someone adds a counter to IndexScanInstrumentation, or a new node type with private statistics or EState-level state. It works everywhere except under GraphScan, where EXPLAIN ANALYZE shows only depth 0's share, or depths disturb each other's state. Nobody notices until a user reports wrong EXPLAIN numbers or wrong rows.

GraphScan runs ExecInitNode on up to about a thousand copies of the inner plan for each execution (messages #9, #11). Message #4 notes that the parameter workspace, SubPlan states and runtime pruning state live on the EState and are shared across copies. Message #12 shows the rollup already misreports Index Searches and double-counts with NO SCROLL cursors. Every future executor node and instrumentation change inherits this, and the current design states no contract for it.

**Question:** Which executor invariants must a node under GraphScan's inner plan respect, and how does a future node type find out it breaks them?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAAAe_zAGW6jPu0y0pX4eKeNBjEB1yQfkDgP4mf4-30Oe1qVu%2BA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAAAe_zBwWs0yC94xN8Vk_8nap0%2B%3Dafs%2BnFhQy5kWUAWhcO_GZA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAAAe_zByDpnT0L1iG_dnneSFjxhq6u9p62Eqmo8uE_1p_jjn4g%40mail.gmail.com); `maintenance-fit-2`</sub>

### 4. Variable-length paths could reuse WITH RECURSIVE instead of a new node

A user writes a quantified GRAPH_TABLE pattern, and it runs through a new GraphScan traversal engine with its own depth limit, frame stack and instrumentation. The same traversal written as WITH RECURSIVE over the edge tables runs through RecursiveUnion, which already offers SEARCH DEPTH FIRST and CYCLE. Fixes and features for path traversal must then be made twice.

Postgres already runs variable-length traversal over tables through recursive queries, with depth-first ordering and cycle detection in the SQL syntax. The proposal does not say why the rewriter cannot expand a VLE element into a recursive subquery, and why that cannot be mixed with fixed-length elements. Message #1 explains only why a custom node could not be mixed with non-VLE patterns. Message #4 favours moving execution out of the rewriter and notes that the same machinery would serve WITH RECURSIVE, which points to overlap with RecursiveUnion rather than a reason against reusing it.

**Question:** Which VLE pattern, mixed with fixed-length elements, cannot be rewritten into a recursive query, or runs unacceptably slowly that way?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAAAe_zBwWs0yC94xN8Vk_8nap0%2B%3Dafs%2BnFhQy5kWUAWhcO_GZA%40mail.gmail.com); `already-exists-1`</sub>

### 5. No shown need to move fixed-length patterns off the rewriter

A user runs a fixed two-hop GRAPH_TABLE query over large tables, and the rewriter today plans it as ordinary joins, possibly a hash join. With enable_native_graphtable on by default, the query instead runs as a GraphScan that probes the inner subplan once per vertex. The user sees the slowdown after upgrading and has to find an undocumented GUC to get the old plan back.

The stated problem in message #1 is that VLE patterns could not be mixed with non-VLE patterns. Message #2 widens the goal to replacing the rewriter for all graph queries, but no material shows that fixed-length pattern users have a problem the join planner does not already solve. The proposal shows no numbers against the rewriter, so every GRAPH_TABLE user takes on a new default plan shape to serve the VLE subset. A committer would likely ask for the native path to be limited to quantified hops until benefit for the rest is measured.

**Question:** For fixed-length patterns, which queries run faster or become possible on the native path, and by how much compared with the rewriter's join plans?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAJyuutg04hOj%3DhQueaKka9-x1BEud41iGyRY5s26tcxUha9rZA%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Does the native path apply the same RLS policies and table privilege checks as the rewriter, and which test shows identical rows for a restricted role?
2. Is the rewriter path for graph queries removed in this series, and if not, what keeps both paths returning the same results?
3. For fixed-length patterns, how do GraphScan plans compare in run time with the rewriter's join plans on large tables?
4. What does the user see when traversal reaches max_graph_stack_depth, an error or a truncated result, and who may change that setting?
5. Why can a VLE element not be expanded into a recursive query that the existing RecursiveUnion node runs?

## Touch points

- **ABI / extension API:** The series adds a GraphScan plan node, changes plannodes.h and execnodes.h, and touches pg_plan_advice's plan walker.
- **SQL-standard conflict or non-standard syntax:** 0003 adds grammar for VLE quantifiers in GRAPH_TABLE, and only path mode WALK is supported.
- **new GUC:** 0002 adds enable_native_graphtable, on by default, and a traversal depth limit.
- **planner behaviour change:** All GRAPH_TABLE queries get a GraphScan plan by default instead of the rewriter's join plans.
- **pg_dump / restore round-trip:** A view with a VLE quantifier must be deparsed by ruleutils.c, and 0003 does not touch that file.
- **security / privileges:** Message #1 says row level security was not started on the native path, which is on by default.
- **maintenance burden ("who maintains this in 10 years"):** A new executor node with per-depth PlanState copies, plus a second execution path beside the rewriter.
- **performance on the hot path:** see objection maintenance-fit-2: Per-depth copies of arbitrary subplans break the one-PlanState-per-Plan assumption

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the executor's PlanState and EState rules and the planner's path and plan creation code, and who follows the SQL/PGQ base.
- Priority: fix design first; audience niche; review cost high. The series is about 300 KiB across parser, planner and executor, serves users of GRAPH_TABLE, and sits on a base that is reverted. The RLS gap, the two-path question and the per-depth PlanState contract need answers before line-by-line review pays off.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
