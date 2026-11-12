# Design pre-review: [PATCH 0/3] Improve UNION de-duplication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7496](https://commitfest.postgresql.org/patch/7496/); [thread](https://www.postgresql.org/message-id/flat/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No surviving objection is fatal, and none fails level 1a or 1b. Three fixable level 2 points remain: standards-compat-1 (single-row locking and fewer volatile evaluations in existing UNION queries), maintenance-fit-2 (the HashAgg gate uses the SetOp estimator and changes existing DISTINCT plans), and maintenance-fit-3 (no planning-time numbers). The thread has one message and no review, so confidence is low.

Not checked: Without the code, nobody can confirm that the constant check matches the DISTINCT rules, that the hook contract with a NULL input relation matches existing stages, or how the gate changes existing DISTINCT plans.

## Prior attempts

- [parallel distinct union and aggregate support patch](https://www.postgresql.org/message-id/flat/2020101922424962544053@sohu.com) (similar idea; Returned with feedback, 2021-11-08). "Many months on and this patch still needs a rebase to apply, and the thread has stalled. I'm marking this Returned with Feedback." ([message](https://www.postgresql.org/message-id/8AF9A4AF-623B-4CEC-BA49-1353377423EF%40yesql.se)) Different now: P2 added new executor nodes (Parallel BatchSort, then Parallel Redistribute) that send tuples to workers. This series adds no executor node: each worker runs a local HashAggregate below Gather and the leader runs a final one. P2 stalled for lack of rebase and review, with no design objection recorded.
- [Revert "Optimize order of GROUP BY keys".](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com) (related; Reverted, 2022-10-03). "The idea of making a cost-based choice of the order of the sorting columns is not fundamentally unsound, but it requires cost information and data statistics that we don't really have." Different now: P5 was reverted because a cost-based choice relied on multi-column distinct estimates that are not trustworthy. This series also picks a plan from a group estimate, but it checks the worst-case worker hash table against hash_mem when the estimate is weak.

## Objections a reviewer is likely to raise

### 1. Implicit LIMIT 1 silently weakens row locking in existing UNION queries

An application runs `select status from (select status from jobs where status = 'pending' for update) s union select 'none'` to lock every pending job. After upgrade, patch 3 adds Limit 1 and only the first row is locked. Concurrent workers then take the same jobs, and the bug appears under load in production with no error or plan warning.

Message #1 says a locking clause in a non-pullable subquery or CTE "locks only the first row" and asks whether children with rowMarks should be excluded. So the point is open. The DISTINCT precedent covers new behaviour that users already accepted, but for UNION this changes what an existing query does after a major-version upgrade, without opt-in. Fewer volatile qual and CTE evaluations change visible side effects in the same way. The material names no release note or user-facing documentation for this change.

**Question:** Why not exclude children with rowMarks or volatile expressions from the implicit LIMIT 1, and what would users lose if they were excluded?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com); `standards-compat-1`</sub>

### 2. HashAgg memory gate borrows the SetOp hash table estimator

A later release changes the SetOp hash table layout and updates EstimateSetOpHashTableSpace(). That edit also moves the gate on partial hash paths for DISTINCT and UNION. Existing DISTINCT queries change plan with no test or reviewer linking the two, and users see this as a regression after upgrade.

Patch 1 changes plan choice for existing SELECT DISTINCT queries, not only for the new UNION path, using EstimateSetOpHashTableSpace() as a "conservative allowance" for HashAgg. Message #1 supports this choice with one near-unique 200k-row case and calls a HashAgg estimator a follow-up. A committer would likely want the gate to use an estimator for the node that actually runs, so changes to one executor node cannot silently move planning for another.

Precedent: [Revert "Optimize order of GROUP BY keys".](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com) (Reverted, 2022-10-03)

**Question:** Why not size the gate with a HashAgg-specific estimate in this series, and what keeps future SetOp changes from shifting DISTINCT plans?

<sub>fixable, level 2, planner behaviour change; messages [#1](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com); `maintenance-fit-2`</sub>

### 3. No planning-time measurement for queries that do not benefit

An OLTP workload runs many short non-ALL UNION queries over small tables. Each query now builds extra partial paths and runs the constant check on every child. Total time rises from planning cost alone, and nobody sees this in EXPLAIN ANALYZE execution numbers.

All benchmarks in message #1 report execution time on 1M-row tables, from one laptop under WSL2 with 4 runs each. Patch 2 builds new paths whenever the input has partial paths, and patch 3 inspects every child of every non-ALL set operation. The proposal reports no planning-time numbers, so the cost for users who gain nothing is unknown.

**Question:** What is the planning-time overhead for short UNION, INTERSECT and EXCEPT queries with many children where neither optimization applies?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/tencent_D56FD01446E8B3D6FA8234F55526DB543F0A%40qq.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Why not exclude set-operation children with rowMarks or volatile expressions from the implicit LIMIT 1, and what speedup would users lose?
2. Why does the hash_mem gate use EstimateSetOpHashTableSpace() and not a HashAgg-specific estimate in this series?
3. Which existing SELECT DISTINCT queries change plan because of the new gate in patch 1, and in which direction?
4. What is the planning-time overhead for short UNION, INTERSECT and EXCEPT queries with many children where neither optimization applies?
5. Does any existing create_upper_paths_hook user misbehave when called with UPPERREL_PARTIAL_SETOP and a NULL input relation?

## Touch points

- **ABI / extension API:** Patch 2 adds UPPERREL_PARTIAL_SETOP to create_upper_paths_hook; patch 1 adds estimate_num_groups_unreliable() and SELFLAG_USED_MULTIVAR_CLAMP.
- **planner behaviour change:** New partial HashAggregate paths for UNION, Limit 1 on constant set-operation children, and a new gate on existing partial DISTINCT hash paths.
- **security / privileges:** Patch 3 can make a locking clause in a non-pullable subquery or CTE lock only the first row.
- **backpatchability:** see objection standards-compat-1: Implicit LIMIT 1 silently weakens row locking in existing UNION queries
- **maintenance burden ("who maintains this in 10 years"):** A new upper stage, a new test module, and a HashAgg memory gate tied to EstimateSetOpHashTableSpace().
- **performance on the hot path:** Patch 3 runs the constant check on every child of every non-ALL set operation at planning time.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows upper-rel planning, set-operation planning in prepunion.c and group-count estimation in selfuncs.c. Nobody has replied in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost high. Plain UNION is common and message #1 reports about 2.6 times speedups for low-cardinality inputs, but only parallel plans and constant children gain. The series is about 140 KiB across planner.c, prepunion.c and selfuncs.c, and no committer has replied.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
