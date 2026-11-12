# Design pre-review: Native planner and executor for SQL/PGQ

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7258](https://commitfest.postgresql.org/patch/7258/) (Needs review); authors: Henri Gasc (henri.gasc)
- Thread: [\[SQL/PGQ\] Native executor for Graph query](https://www.postgresql.org/message-id/flat/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), started 2026-08-08
- Messages read: 12 of 12
- Patch prose read: v2-0002-Add-GUC-enable_native-and-max_depth.patch, v2-0001-Restore-SQL-PGQ-as-a-cfbot-base-not-for-review.patch, v2-0005-Implement-the-executor.patch, v2-0003-Parse-grammar-for-VLE.patch, v2-0008-Remove-dead-code-and-simplify-GraphScan.patch, v2-0007-Add-rows-estimate-to-planner-and-use-IndexScan-if.patch, v2-0004-Split-graph-queries-into-GraphScan-Append-SeqScan.patch, v2-0006-Correct-bug-const-with-label-disjunction-and-show.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal of running VLE patterns is reasonable, and no surviving objection is fatal. The design leaves open questions that the list must settle before code review. The native path is on by default while message #1 says row level security was not started (security-1). The series keeps two execution paths behind a GUC (maintenance-fit-1) and moves fixed-length patterns off the join planner without numbers (demand-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, RLS enforcement in the inner subplans, deparsing of VLE quantifiers, and the context and upper bound of the depth GUC could not be checked. Nine of the 12 thread messages were seen only through the objections.

## The proposal, restated

**Problem.** Graph queries in SQL/PGQ are run by rewriting them into relational joins, and a custom node for variable-length edge (VLE) patterns could not be mixed with non-VLE patterns in one query. The author wants graph patterns, including VLE, planned and executed natively for users of GRAPH_TABLE.

**Proposed design.** A new GraphScan plan node handles one quantified hop, with a parameterized inner subplan (Append over SeqScan or IndexScan per element table) that expands one hop from the current vertex. The executor walks the graph depth first with per-depth state and backtracking, and only path mode WALK is supported. The series is meant to replace the rewriter for graph queries, and it sits on a 0001 commit that restores the reverted SQL/PGQ base.

**User-visible surface.**
- GUC enable_native_graphtable, on by default; off uses the rewriter.
- GUC limiting traversal depth (max_graph_stack_depth, default 1001 per the reviewer; the commit title says max_depth).
- Grammar for VLE quantifiers inside GRAPH_TABLE MATCH patterns.
- New GraphScan node in EXPLAIN output, with the inner subplan shown under "Inner".
- For a lateral query such as select * from x1, graph_table (...), the author first proposed to reject it; v2 instead matches the rewriter's results.

**What the author claims.**
- The native path supports partitioning, inheritance, foreign tables and views, and lateral queries give the same results as the rewriter.
- Row level security was not handled in the first version, and later messages do not say it is handled now.
- The planner estimates rows and uses IndexScan for the inner subplan when possible.
- Future work could add other path modes, BFS or bidirectional search, and predicate pushdown.

**Thread state.** Henson Choi reviewed it and proposed that per-depth PlanState copies be created lazily and linked through up/down pointers on PlanState, sending WIP patches. The author prefers to commit a working version first; the instrumentation rollup and eager init of about a thousand frames per GraphScan are still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Native path on by default without row level security enforcement

fixable · level 2, rubric item 7 · security / privileges, new GUC · `security-1`

A DBA puts RLS policies on an edge or vertex table and grants SELECT on a property graph over it. A restricted user runs GRAPH_TABLE with a VLE pattern; the native GraphScan path, enabled by default, may return rows the policy hides. Nothing reports an error, so the leak goes unnoticed until someone audits the results.

Message #1 says row level security was not approached at all. Messages #2 to #12 add partitioning, views, foreign tables and lateral handling but never say RLS is enforced. The rewriter gets RLS and per-table permission checks from ordinary relational range table entries. GraphScan builds its own inner subplans per element table, and the proposal does not say those subplans still carry the RLS quals and permission checks while enable_native_graphtable defaults to on.

Thread: [#1 (Henri GASC, 2026-08-08)](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2 (Henri GASC, 2026-09-04)](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com), [#4 (Henson Choi, 2026-09-07)](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#7 (Henri GASC, 2026-09-15)](https://www.postgresql.org/message-id/CAJyuutgJzqVJEsKGFaQpPbp812TcAx69wpgSuHjn6vVDmEdBVw%40mail.gmail.com)

**Question:** Does the native path apply the same RLS policies and table privilege checks as the rewriter, with tests that show identical rows for a restricted role?

Counter-check: held. Message #1 says row level security was not started, and no later message says it is enforced. The native path is on by default, so the gap would reach users who never set the GUC.

### 2. Two graph execution paths with a GUC that switches between them

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), new GUC, planner behaviour change · `maintenance-fit-1`

A later patch fixes a GRAPH_TABLE semantics bug or adds a pattern feature in rewriteGraphTable.c and does not touch nodeGraphScan.c. Results now depend on enable_native_graphtable. A user finds out when the two settings return different rows for the same query.

Message #2 says the series aims to replace the rewriter completely, yet enable_native_graphtable keeps the rewriter as a live, user-selectable path. The two paths already diverged once: v2 had to copy the rewriter's lateral results instead of the documented Cartesian product (messages #1, #2). Nothing makes a change to one path fail when the other is not updated. The proposal does not say when the rewriter path goes away, or how the two are kept in agreement until then.

Thread: [#1 (Henri GASC, 2026-08-08)](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2 (Henri GASC, 2026-09-04)](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com)

**Question:** Is the rewriter path removed in this series, and if not, what keeps both paths returning the same results until it is?

Counter-check: held. Message #2 states the aim to replace the rewriter, yet the series keeps both paths behind a GUC. No message says when the rewriter path is removed or what test keeps the two in agreement.

### 3. Per-depth copies of arbitrary subplans break the one-PlanState-per-Plan assumption

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, performance on the hot path · `maintenance-fit-2`

Someone adds a counter to IndexScanInstrumentation, or a new node type with private statistics or EState-level state. It works everywhere except under GraphScan, where EXPLAIN ANALYZE shows only depth 0's share, or depths disturb each other's state. Nobody notices until a user reports wrong EXPLAIN numbers or wrong rows.

GraphScan runs ExecInitNode on up to about a thousand copies of the inner plan for each execution (messages #9, #11). Message #4 notes that the parameter workspace, SubPlan states and runtime pruning state live on the EState and are shared across copies. Message #12 shows the rollup already misreports Index Searches and double-counts with NO SCROLL cursors. Every future executor node and instrumentation change inherits this, and the current design states no contract for it.

Thread: [#4 (Henson Choi, 2026-09-07)](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#9 (Henson Choi, 2026-09-17)](https://www.postgresql.org/message-id/CAAAe_zAGW6jPu0y0pX4eKeNBjEB1yQfkDgP4mf4-30Oe1qVu%2BA%40mail.gmail.com), [#11 (Henson Choi, 2026-09-17)](https://www.postgresql.org/message-id/CAAAe_zBwWs0yC94xN8Vk_8nap0%2B%3Dafs%2BnFhQy5kWUAWhcO_GZA%40mail.gmail.com), [#12 (Henson Choi, 2026-09-17)](https://www.postgresql.org/message-id/CAAAe_zByDpnT0L1iG_dnneSFjxhq6u9p62Eqmo8uE_1p_jjn4g%40mail.gmail.com)

**Question:** Which executor invariants must a node under GraphScan's inner plan respect, and how does a future node type find out it breaks them?

Counter-check: held. Messages #4 and #9 support the shared EState concern and the frame 0 only EXPLAIN output. The Index Searches and NO SCROLL issues in message #12 belong to the reviewer's WIP patch, so they must not be counted against the series.

### 4. Variable-length paths could reuse WITH RECURSIVE instead of a new node

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years"), planner behaviour change · `already-exists-1`

A user writes a quantified GRAPH_TABLE pattern, and it runs through a new GraphScan traversal engine with its own depth limit, frame stack and instrumentation. The same traversal written as WITH RECURSIVE over the edge tables runs through RecursiveUnion, which already offers SEARCH DEPTH FIRST and CYCLE. Fixes and features for path traversal must then be made twice.

Postgres already runs variable-length traversal over tables through recursive queries, with depth-first ordering and cycle detection in the SQL syntax. The proposal does not say why the rewriter cannot expand a VLE element into a recursive subquery, and why that cannot be mixed with fixed-length elements. Message #1 explains only why a custom node could not be mixed with non-VLE patterns. Message #4 favours moving execution out of the rewriter and notes that the same machinery would serve WITH RECURSIVE, which points to overlap with RecursiveUnion rather than a reason against reusing it.

Thread: [#1 (Henri GASC, 2026-08-08)](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#4 (Henson Choi, 2026-09-07)](https://www.postgresql.org/message-id/CAAAe_zBVM-xhJvf5cbmocsPQFD9OC1MNPJmaQY05U66NohaRuQ%40mail.gmail.com), [#11 (Henson Choi, 2026-09-17)](https://www.postgresql.org/message-id/CAAAe_zBwWs0yC94xN8Vk_8nap0%2B%3Dafs%2BnFhQy5kWUAWhcO_GZA%40mail.gmail.com)

**Question:** Which VLE pattern, mixed with fixed-length elements, cannot be rewritten into a recursive query, or runs unacceptably slowly that way?

Counter-check: held. Message #1 explains only why a custom node failed, and no message compares a rewrite to a recursive query. Message #4 says moving execution out of the rewriter was a direction given earlier, but the material does not show that reasoning.

### 5. No shown need to move fixed-length patterns off the rewriter

fixable · level 1b, rubric item 4 · desirability, planner behaviour change, new GUC, maintenance burden ("who maintains this in 10 years") · `demand-1`

A user runs a fixed two-hop GRAPH_TABLE query over large tables, and the rewriter today plans it as ordinary joins, possibly a hash join. With enable_native_graphtable on by default, the query instead runs as a GraphScan that probes the inner subplan once per vertex. The user sees the slowdown after upgrading and has to find an undocumented GUC to get the old plan back.

The stated problem in message #1 is that VLE patterns could not be mixed with non-VLE patterns. Message #2 widens the goal to replacing the rewriter for all graph queries, but no material shows that fixed-length pattern users have a problem the join planner does not already solve. The proposal shows no numbers against the rewriter, so every GRAPH_TABLE user takes on a new default plan shape to serve the VLE subset. A committer would likely ask for the native path to be limited to quantified hops until benefit for the rest is measured.

Thread: [#1 (Henri GASC, 2026-08-08)](https://www.postgresql.org/message-id/CAJyuutjsZbTVg4O1zxr9%3Dvwk%3DiKkbXtq9wCW6PhBVaXg%3DDM6Qg%40mail.gmail.com), [#2 (Henri GASC, 2026-09-04)](https://www.postgresql.org/message-id/CAJyuutgjpVNhm8b%2BhgqJEr_7Q-VsOjAeM1vKr3sM1GGZjVBU8g%40mail.gmail.com), [#10 (Henri GASC, 2026-09-17)](https://www.postgresql.org/message-id/CAJyuutg04hOj%3DhQueaKka9-x1BEud41iGyRY5s26tcxUha9rZA%40mail.gmail.com)

**Question:** For fixed-length patterns, which queries run faster or become possible on the native path, and by how much compared with the rewriter's join plans?

Counter-check: held. The series shows no numbers against the rewriter, and message #2 widens the goal from VLE to all graph queries. A per-vertex probe replacing a join plan chosen by the planner is a concrete regression risk for fixed-length patterns.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series adds a GraphScan plan node, changes plannodes.h and execnodes.h, and touches pg_plan_advice's plan walker. |
| SQL-standard conflict or non-standard syntax | **yes** | 0003 adds grammar for VLE quantifiers in GRAPH_TABLE, and only path mode WALK is supported. |
| new GUC | **yes** | 0002 adds enable_native_graphtable, on by default, and a traversal depth limit. |
| planner behaviour change | **yes** | All GRAPH_TABLE queries get a GraphScan plan by default instead of the rewriter's join plans. |
| pg_dump / restore round-trip | **yes** | A view with a VLE quantifier must be deparsed by ruleutils.c, and 0003 does not touch that file. |
| psql UX conventions | no |  |
| security / privileges | **yes** | Message #1 says row level security was not started on the native path, which is on by default. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new executor node with per-depth PlanState copies, plus a second execution path beside the rewriter. |
| performance on the hot path | **yes** | see objection maintenance-fit-2: Per-depth copies of arbitrary subplans break the one-PlanState-per-Plan assumption |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the native path apply the same RLS policies and table privilege checks as the rewriter, and which test shows identical rows for a restricted role?
2. Is the rewriter path for graph queries removed in this series, and if not, what keeps both paths returning the same results?
3. For fixed-length patterns, how do GraphScan plans compare in run time with the rewriter's join plans on large tables?
4. What does the user see when traversal reaches max_graph_stack_depth, an error or a truncated result, and who may change that setting?
5. Why can a VLE element not be expanded into a recursive query that the existing RecursiveUnion node runs?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the executor's PlanState and EState rules and the planner's path and plan creation code, and who follows the SQL/PGQ base.
- **Liability:** The owner signs for a new plan node, its per-depth copies of arbitrary inner plans, two GUCs, and agreement between two graph execution paths for as long as both exist. The series also depends on a reverted base that message #6 expects to be re-posted in another form.
- **Author follow-up:** The author posted updates in messages #2 and #3 and a split v2 series in message #8, and answers the reviewer. The author prefers to commit a working version before changing frame init to run on first use.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The series is about 300 KiB across parser, planner and executor, serves users of GRAPH_TABLE, and sits on a base that is reverted. The RLS gap, the two-path question and the per-depth PlanState contract need answers before line-by-line review pays off.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Unbounded quantifiers under WALK are cut off by a GUC. Not listed: not among the strongest objections.
- `replication-ondisk-1` (fixable, level 2) Views using VLE quantifiers may not survive pg_dump and pg_upgrade. Not listed: not among the strongest objections.
- `security-2` (fixable, level 2) Traversal depth setting lets any user drive per-query memory. Not listed: not among the strongest objections.
- `maintenance-fit-3` (fixable, level 2) Large new planner and executor area, no committer engaged, base reverted. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-1` Default-on GUC picks between two executors with possibly different results. Refuted (not specific): Duplicate of security-1 for the row level security gap, and of maintenance-fit-1 for results that depend on the GUC. Those two state the same points with more detail.
- `already-exists-2` Two execution paths for the same graph queries behind a GUC. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about two paths behind enable_native_graphtable and asks the same question.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and nothing in the thread points to an earlier rejection that this series repeats. The thread mentions two earlier events: the author's own custom VLE node (message #1), which this series replaces because that node could not mix VLE and non-VLE patterns, and the SQL/PGQ base revert (messages #5, #6). Message #6 reports that the revert was about catalogs and dependencies, not execution, so its reason does not apply to this executor design. The series still cannot go in until that base returns.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. pg_dump is not affected, because the series adds no new objects beyond the restored SQL/PGQ base.
- replication-ondisk: 1 objection(s). Apart from view deparsing, the series adds no on-disk format, WAL record, wire protocol message or replicated state. The catalog changes belong to the restored SQL/PGQ base in 0001, which is marked not for review, and GraphScan state lives only within one execution.
- security: 2 objection(s). Two objections from the security angle; no listed precedent applies.
- maintenance-fit: 3 objection(s). Objections listed above. Hot-path cost for queries that do not use GRAPH_TABLE looks nil from the prose, so no objection is raised on that point.
- already-exists: 2 objection(s). No contrib module or extension that already provides SQL/PGQ graph execution is named in the material, so no objection rests on an external alternative.
- demand: 1 objection(s). Demand for VLE itself is reasonable: quantified path patterns are part of SQL/PGQ, and a reviewer in message #4 supports the direction. The only objection concerns widening the change to all graph queries.

### Precedents retrieved

- P1: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P2: [Make executor's Range Table an array instead of a List](https://commitfest.postgresql.org/patch/1771/) (cf, Returned with feedback, 2018-08-23) - judged not relevant; reason summarised from its thread
- P3: [Allow LISTEN on patterns](https://commitfest.postgresql.org/patch/5630/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P4: [Revert "Move PartitionPruneInfo out of plan nodes into PlannedStmt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5472743d9e8583638a897b47558066167cc14583) (revert, Reverted, 2023-05-04) - judged not relevant
- P5: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P6: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P7: [JIT compilation per plan node](https://commitfest.postgresql.org/patch/4837/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P8: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3560/) (cf, Returned with feedback, 2022-02-26) - judged not relevant
- P9: [Revert "Don't lock partitions pruned by initial pruning"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1722d5eb05d8e5d2e064cd1798abcae4f296ca9d) (revert, Reverted, 2025-05-22) - judged not relevant
- P10: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P11: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3561/) (cf, Returned with feedback, 2022-12-01) - judged not relevant
- P12: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant; reason summarised from its thread
- P13: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P14: [Add RESPECT/IGNORE NULLS and FROM FIRST/LAST options](https://commitfest.postgresql.org/patch/5852/) (cf, Withdrawn, 2025-06-26) - judged not relevant; reason summarised from its thread
- P15: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant; reason summarised from its thread
- P16: [Pattern based listeners for asynchronous messaging (LISTEN/NOTIFY)](https://commitfest.postgresql.org/patch/1218/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P17: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P18: [Autoprepare: implicitly replace literals with parameters and store generalized plan](https://commitfest.postgresql.org/patch/1747/) (cf, Returned with feedback, 2020-07-01) - judged not relevant; reason summarised from its thread
- P19: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/4309/) (cf, Returned with feedback, 2023-09-19) - judged not relevant
- P20: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant
- P21: [Exclude partition constaint checking in query execution plan for partitioned table queries](https://commitfest.postgresql.org/patch/1264/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Revert inappropriate weakening of an Assert in plpgsql.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=16a3ae504e2030583851c8f45cbed14b0c546d6e) (revert, Reverted, 2025-03-21) - judged not relevant
- P23: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant
- P24: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant

### Warnings

- attachment limit (8) reached; graph_table_hops.txt not read
- attachment limit (8) reached; native_graph_20260808.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 210 s; cost reported by the CLI: $3.28
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 106 text field(s) checked, 3 finding(s), 3 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:34:23.009Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
