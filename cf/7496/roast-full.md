# Design pre-review: [PATCH 0/3] Improve UNION de-duplication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7496](https://commitfest.postgresql.org/patch/7496/) (Needs review); authors: Xiangxin Zeng (xiangxin-zeng)
- Thread: [\[PATCH 0/3\] Improve UNION de-duplication](https://www.postgresql.org/message-id/flat/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com), started 2026-10-22
- Messages read: 1 of 1
- Patch prose read: 0001-Extract-shared-de-duplication-helpers-and-gate-DISTI.patch, 0002-Add-parallel-partial-deduplication-for-UNION.patch, 0003-Add-implicit-LIMIT-1-for-constant-set-operation-chil.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No surviving objection is fatal, and none fails level 1a or 1b. Three fixable level 2 points remain: standards-compat-1 (single-row locking and fewer volatile evaluations in existing UNION queries), maintenance-fit-2 (the HashAgg gate uses the SetOp estimator and changes existing DISTINCT plans), and maintenance-fit-3 (no planning-time numbers). The thread has one message and no review, so confidence is low.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that the constant check matches the DISTINCT rules, that the hook contract with a NULL input relation matches existing stages, or how the gate changes existing DISTINCT plans.

## The proposal, restated

**Problem.** For low-cardinality inputs, a parallel hash-based UNION does all de-duplication in one HashAggregate above Gather, so the leader reads every row from every child. A set-operation child whose output is one constant tuple still scans all matching rows. DISTINCT already handles both cases, so UNION queries are slower than the equivalent DISTINCT over UNION ALL.

**Proposed design.** Patch 1 moves the shared DISTINCT/setop path construction into planner.c helpers and adds a hash_mem check for the partial DISTINCT hash path when the group estimate is unreliable. Patch 2 adds a worker-local HashAggregate below Gather for UNION through a new UPPERREL_PARTIAL_SETOP stage, with the final HashAggregate in the leader. Patch 3 adds LIMIT 1 to constant children of non-ALL UNION, INTERSECT and EXCEPT, including partial paths.

**User-visible surface.**
- New upper relation stage UPPERREL_PARTIAL_SETOP, passed to create_upper_paths_hook with a NULL input relation; it may be called more than once per query.
- New selfuncs C API: estimate_num_groups_unreliable() and the SELFLAG_USED_MULTIVAR_CLAMP flag.
- New plan shapes in EXPLAIN: partial HashAggregate under Gather for UNION, and Limit 1 on constant set-operation children.
- Fewer evaluations of volatile quals, FROM items or CTEs in constant set-operation children, and locking clauses in a non-pullable subquery or CTE may lock only the first row.

**What the author claims.**
- Low-cardinality UNION of 2 to 4 tables of 1M rows runs about 2.6 to 2.8 times faster.
- A constant branch plus a varying branch runs about 4 times faster in parallel and 1.7 times faster serially.
- Near-unique keys, UNION ALL, DISTINCT, INTERSECT, EXCEPT and grouped-child queries show no change within noise.
- The early-stopping behavior matches existing SELECT DISTINCT LIMIT 1 behavior, so it has precedent.
- make check, make check-world and a cassert Meson build pass.

**Thread state.** Only the first message is visible; nobody has replied. The author asks whether early stopping with locking clauses is acceptable for set operations, or whether children with rowMarks should be excluded.

## 2. Prior attempts

### parallel distinct union and aggregate support patch (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2769/](https://commitfest.postgresql.org/patch/2769/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2020101922424962544053@sohu.com](https://www.postgresql.org/message-id/flat/2020101922424962544053@sohu.com)
- **Date:** decided 2021-11-08; started 2020-10-19
- **Outcome:** Returned with feedback (cf)
- **Who objected:** David Steele, David Rowley, Daniel Gustafsson (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The entry was returned with feedback because the patch needed a rebase for many months and the thread stalled. It got no review for two commitfests, and David Steele and David Rowley said the author had not answered reviewers who suggested an unbuffered Parallel Partition style operator. The author posted a Redistribute version in September 2021, but the visible messages show no review of it.
- **Reason (quote):** "Many months on and this patch still needs a rebase to apply, and the thread has stalled. I'm marking this Returned with Feedback." ([message](https://www.postgresql.org/message-id/8AF9A4AF-623B-4CEC-BA49-1353377423EF%40yesql.se))
- **What is different this time:** P2 added new executor nodes (Parallel BatchSort, then Parallel Redistribute) that send tuples to workers. This series adds no executor node: each worker runs a local HashAggregate below Gather and the leader runs a final one. P2 stalled for lack of rebase and review, with no design objection recorded.

### Revert "Optimize order of GROUP BY keys". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3)
- **Thread:** [https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com), [https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6@sigaev.ru](https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6@sigaev.ru), [https://www.postgresql.org/message-id/flat/CA+q6zcW_4o2NC0zutLkOJPsFt80megSpX_dVRo6GK9PC-Jx_Ag@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+q6zcW_4o2NC0zutLkOJPsFt80megSpX_dVRo6GK9PC-Jx_Ag@mail.gmail.com)
- **Date:** decided 2022-10-03; started 2022-03-30
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit db0d67db2401eb6238ccc04c6407a4fd4f985832 and several follow-on fixes. The idea of making a cost-based choice of the order of the sorting columns is not fundamentally unsound, but it requires cost information and data statistics that we don't really have. For example, relying on procost to distinguish the relative costs of different sort comparators is pretty pointless so long as most such comparator functions are labeled with cost 1.0. Moreover, estimating the number of comparisons done by Quicksort requires more than just an estimate of the number of distinct values in the input: you also need some idea of the sizes of the larger groups, if you want an estimate that's good to better than a factor of three or so. That's data that's often unknown or not very reliable. Worse, to arrive at estimates of the number of calls made to the lower-order-column comparison functions, the code needs to make estimates of the numbers of distinct values of multiple columns, which are necessarily even less trustworthy than per-column stats. Even if all the inputs are perfectly reliable, the cost algorithm as-implemented cannot offer useful information about how to order sorting columns beyond the point at which the average group size is estimated to drop to 1. Close inspection of the code added by db0d67db2 shows that there are also multiple small bugs. These could have been fixed, but there's not much point if we don't trust the estimates to be accurate in-principle. Finally, the changes in cost_sort's behavior made for very large changes (often a factor of 2 or so) in the cost estimates for all sorting operations, not only those for multi-column GROUP BY. That naturally changes plan choices in many situations, and there's precious little evidence to show that the changes are for the better. Given the above doubts about whether the new estimates are really trustworthy, it's hard to summon much confidence that these changes are better on the average. Since we're hard up against the release deadline for v15, let's revert these changes for now. We can always try again later. Note: in v15, I left T_PathKeyInfo in place in nodes.h even though it's unreferenced. Removing it would be an ABI break, and it seems a bit late in the release cycle for that.
- **Reason (quote):** "The idea of making a cost-based choice of the order of the sorting columns is not fundamentally unsound, but it requires cost information and data statistics that we don't really have."
- **What is different this time:** P5 was reverted because a cost-based choice relied on multi-column distinct estimates that are not trustworthy. This series also picks a plan from a group estimate, but it checks the worst-case worker hash table against hash_mem when the estimate is weak.


## 3. Objections a reviewer is likely to raise

### 1. Implicit LIMIT 1 silently weakens row locking in existing UNION queries

fixable · level 2, rubric item 7 · security / privileges, planner behaviour change, backpatchability · `standards-compat-1`

An application runs `select status from (select status from jobs where status = 'pending' for update) s union select 'none'` to lock every pending job. After upgrade, patch 3 adds Limit 1 and only the first row is locked. Concurrent workers then take the same jobs, and the bug appears under load in production with no error or plan warning.

Message #1 says a locking clause in a non-pullable subquery or CTE "locks only the first row" and asks whether children with rowMarks should be excluded. So the point is open. The DISTINCT precedent covers new behaviour that users already accepted, but for UNION this changes what an existing query does after a major-version upgrade, without opt-in. Fewer volatile qual and CTE evaluations change visible side effects in the same way. The material names no release note or user-facing documentation for this change.

Thread: [#1 (zengxx, 2026-10-22)](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com)

**Question:** Why not exclude children with rowMarks or volatile expressions from the implicit LIMIT 1, and what would users lose if they were excluded?

Counter-check: held. Message #1 confirms single-row locking and fewer volatile evaluations, and asks whether children with rowMarks should be excluded. DISTINCT's Limit 1 also changed existing queries on upgrade, so that contrast is weaker than stated, but the open question stands.

### 2. HashAgg memory gate borrows the SetOp hash table estimator

fixable · level 2, rubric item 9 · planner behaviour change, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later release changes the SetOp hash table layout and updates EstimateSetOpHashTableSpace(). That edit also moves the gate on partial hash paths for DISTINCT and UNION. Existing DISTINCT queries change plan with no test or reviewer linking the two, and users see this as a regression after upgrade.

Patch 1 changes plan choice for existing SELECT DISTINCT queries, not only for the new UNION path, using EstimateSetOpHashTableSpace() as a "conservative allowance" for HashAgg. Message #1 supports this choice with one near-unique 200k-row case and calls a HashAgg estimator a follow-up. A committer would likely want the gate to use an estimator for the node that actually runs, so changes to one executor node cannot silently move planning for another.

**Precedents.**
- [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (Reverted, 2022-10-03); thread: [#1](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com), [#2](https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6@sigaev.ru), [#3](https://www.postgresql.org/message-id/flat/CA+q6zcW_4o2NC0zutLkOJPsFt80megSpX_dVRo6GK9PC-Jx_Ag@mail.gmail.com)

Thread: [#1 (zengxx, 2026-10-22)](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com)

**Question:** Why not size the gate with a HashAgg-specific estimate in this series, and what keeps future SetOp changes from shifting DISTINCT plans?

Counter-check: held. Patch 1 changes plan choice for existing DISTINCT queries using a SetOp estimator, and message #1 supports this with one 200k-row case. P5 concerns unreliable statistics, so it does not support the point about coupling two executor nodes.

### 3. No planning-time measurement for queries that do not benefit

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `maintenance-fit-3`

An OLTP workload runs many short non-ALL UNION queries over small tables. Each query now builds extra partial paths and runs the constant check on every child. Total time rises from planning cost alone, and nobody sees this in EXPLAIN ANALYZE execution numbers.

All benchmarks in message #1 report execution time on 1M-row tables, from one laptop under WSL2 with 4 runs each. Patch 2 builds new paths whenever the input has partial paths, and patch 3 inspects every child of every non-ALL set operation. The proposal reports no planning-time numbers, so the cost for users who gain nothing is unknown.

Thread: [#1 (zengxx, 2026-10-22)](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com)

**Question:** What is the planning-time overhead for short UNION, INTERSECT and EXCEPT queries with many children where neither optimization applies?

Counter-check: held. Message #1 reports only execution time. Small tables get no partial paths, so patch 2 adds little there, but patch 3 runs the constant check on every child of every non-ALL set operation and that cost is unmeasured.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 2 adds UPPERREL_PARTIAL_SETOP to create_upper_paths_hook; patch 1 adds estimate_num_groups_unreliable() and SELFLAG_USED_MULTIVAR_CLAMP. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | New partial HashAggregate paths for UNION, Limit 1 on constant set-operation children, and a new gate on existing partial DISTINCT hash paths. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Patch 3 can make a locking clause in a non-pullable subquery or CTE lock only the first row. |
| backpatchability | **yes** | see objection standards-compat-1: Implicit LIMIT 1 silently weakens row locking in existing UNION queries |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new upper stage, a new test module, and a HashAgg memory gate tied to EstimateSetOpHashTableSpace(). |
| performance on the hot path | **yes** | Patch 3 runs the constant check on every child of every non-ALL set operation at planning time. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why not exclude set-operation children with rowMarks or volatile expressions from the implicit LIMIT 1, and what speedup would users lose?
2. Why does the hash_mem gate use EstimateSetOpHashTableSpace() and not a HashAgg-specific estimate in this series?
3. Which existing SELECT DISTINCT queries change plan because of the new gate in patch 1, and in which direction?
4. What is the planning-time overhead for short UNION, INTERSECT and EXCEPT queries with many children where neither optimization applies?
5. Does any existing create_upper_paths_hook user misbehave when called with UPPERREL_PARTIAL_SETOP and a NULL input relation?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows upper-rel planning, set-operation planning in prepunion.c and group-count estimation in selfuncs.c. Nobody has replied in the visible thread.
- **Liability:** The owner signs for plan changes in existing DISTINCT and UNION queries, for the changed locking and volatile-evaluation behaviour, and for a hook stage that extensions may code against, across all future major versions.
- **Author follow-up:** No sign either way; the thread has one message, in which the author asks an open design question and names a possible follow-up estimator.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Plain UNION is common and message #1 reports about 2.6 times speedups for low-cardinality inputs, but only parallel plans and constant children gain. The series is about 140 KiB across planner.c, prepunion.c and selfuncs.c, and no committer has replied.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` New public upper stage and selfuncs API shaped for one plan shape. Refuted (no failure scenario): Postgres already passes its other partial upper stages to create_upper_paths_hook, and selfuncs already has SELFLAG_ flags, so the patch copies existing practice. An extension that switches on the stage ignores the new value, so the wrong-plan scenario needs code written for the new stage.
- `already-exists-1` Second partial de-duplication path beside DISTINCT's existing one. Refuted (already handled): Message #1 gives the reasons for not reusing the DISTINCT code: varno-0 Vars, no statistics, child-derived group counts and setop-specific MergeAppend. Postgres already plans UNION separately in prepunion.c, and patch 1 moves the shared path construction into common helpers.
- `demand-1` Constant-child LIMIT 1 targets a query shape with no shown users. Refuted (not specific): Duplicate of standards-compat-1: the failure scenario is the same locking change. The remaining demand point is weak because Postgres already applies this optimization to SELECT DISTINCT, and the other costs fall only on queries with a constant child.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected for a reason that still applies. P1, P2 and P3 stalled without a design objection. The P5 concern about unreliable group estimates is addressed in message #1 by the hash_mem gate, which follows the existing partial DISTINCT practice.
- standards-compat: 1 objection(s). The new UPPERREL_PARTIAL_SETOP hook call with a NULL input relation follows how the setop stage is already exposed, so I raise no compatibility objection there. Patches 1 and 2 add no syntax, GUC or dump-visible state.
- replication-ondisk: 0 objection(s). The series changes only the planner. It adds no on-disk format, WAL record, catalog column, wire message or replication behavior, so pg_upgrade, standbys and subscribers are unaffected. The UPPERREL_PARTIAL_SETOP enum value and the new selfuncs symbols land in a new major version only, where extensions recompile anyway. Passing a NULL input relation to create_upper_paths_hook appears to follow how the existing UPPERREL_SETOP stage calls the hook, but this cannot be confirmed without the code. Fewer row locks in hidden locking clauses is a semantics question for another angle.
- security: 0 objection(s). The proposal adds no privilege, role or exposure surface: workers run with the leader's privileges, RLS security quals still apply below the new Limit 1, and per-worker HashAgg memory stays bounded by hash_mem with spilling. Single-row locking in hidden FOR UPDATE subqueries is a concurrency semantics question for another angle.
- maintenance-fit: 3 objection(s). Not applicable; the objections above are fixable design points. No committer has replied in the visible thread, so nobody has yet taken ownership of the planner changes.
- already-exists: 1 objection(s). The constant-child LIMIT 1 has no existing equivalent for set operations, so this angle raises nothing against patch 3.
- demand: 1 objection(s). Patch 2 (parallel partial de-duplication for UNION) targets a common construct and shows about 2.6 times speedups. A planner improvement for plain UNION needs no extra demand evidence, so I raise no demand objection against it.

### Precedents retrieved

- P1: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P2: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - used in the review; reason summarised from its thread
- P3: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - used in the review; reason summarised from its thread
- P4: [Union Replacement of OR logic](https://commitfest.postgresql.org/patch/4715/) (cf, Rejected, 2024-07-24) - judged not relevant; reason summarised from its thread
- P5: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - used in the review
- P6: [(ALTER SYSTEM) SET +=/-= operator support](https://commitfest.postgresql.org/patch/2750/) (cf, Returned with feedback, 2021-03-12) - judged not relevant; reason summarised from its thread
- P7: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P8: [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://commitfest.postgresql.org/patch/7034/) (cf, Returned with feedback, 2026-09-10) - judged not relevant; reason summarised from its thread
- P9: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4019/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P10: [Corresponding by clause for set operation](https://commitfest.postgresql.org/patch/1016/) (cf, Returned with feedback, 2017-04-04) - judged not relevant
- P11: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - used in the review
- P12: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4857/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P13: [Revert "Make some use of anonymous unions \[pgcrypto\]"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5aabe6d58e08179cc656539c4bd0145525bf22f) (revert, Reverted, 2025-09-30) - judged not relevant
- P14: [Partial sort](https://commitfest.postgresql.org/patch/409/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P15: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P16: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P17: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P18: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant
- P19: [Remove 1MB size limit for tsvector](https://commitfest.postgresql.org/patch/1221/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P20: [SLOPE - Planner optimizations on monotonic expressions.](https://commitfest.postgresql.org/patch/6653/) (cf, Withdrawn, 2026-07-20) - judged not relevant
- P21: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P22: [Skip a redundant singleton GROUP BY node](https://commitfest.postgresql.org/patch/7419/) (cf, Withdrawn, 2026-10-09) - judged not relevant
- P23: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P24: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 132 s; cost reported by the CLI: $1.48
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:01:07.675Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
