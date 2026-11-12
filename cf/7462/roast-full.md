# Design pre-review: Improve ndistinct estimates for flattened UNION

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7462](https://commitfest.postgresql.org/patch/7462/) (Needs review); authors: Xiangxin Zeng (xiangxin-zeng)
- Thread: [\[PATCH 0/2\] Improve ndistinct estimates for flattened UNION ALL](https://www.postgresql.org/message-id/flat/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com), started 2026-10-15
- Messages read: 3 of 3
- Patch prose read: 0001-estimate-UNION-ALL-parent-ndistinct-ends.patch, 0002-use-safe-ends-of-UNION-ALL-ndistinct-estimates.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Nothing that survived blocks the idea. The parent columns of a flattened UNION ALL get the default of 200 today, and the cost falls only on queries of that shape. Two fixable points remain: demand-1 (no overlapping-arm case is measured, so hi may overstate group counts) and maintenance-fit-1 (a new caller of get_variable_numdistinct() gets lo without choosing it). A planner committer has to judge the lo/hi policy.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. This review could not check how the lo/hi choice reaches each caller, whether the member lookups keep the usual statistics privilege checks, or what the on-demand traversal costs for wide unions.

## The proposal, restated

**Problem.** After a simple UNION ALL is pulled up into an appendrel, the parent columns have no pg_statistic entry, so most ndistinct lookups return DEFAULT_NUM_DISTINCT (200). A union of millions of keys then looks like 200 distinct values, which skews semi/anti join selectivity, GROUP BY/DISTINCT group counts and the decision to use Memoize.

**Proposed design.** The planner lazily derives two ends from member estimates when a consumer asks about a parent column: lo is the largest member estimate, hi is the sum of all members, or zero if any member is not estimable. Divisor-style callers of get_variable_numdistinct() use lo, while group counts, Memoize and the inner side of equality semi/anti joins use hi, with fallback to the old default or heuristic when hi is unavailable. Only flattened simple UNION ALL parents change.

**What the author claims.**
- Estimates were no worse than baseline in 18 of 18 A/B scenarios: 9 improved, 9 unchanged.
- Median speedups of about 1.2x to 2.1x on semi join, anti join and Memoize cases.
- No statistically significant regression; planning-time increases in 64-arm stress cases had confidence intervals crossing zero.
- A LATERAL Top-N reproducer picks Memoize and does 20 index searches instead of 20000.
- After review, the author restated the Memoize goal: allow Memoize where the default estimate now blocks it, not prevent wrong Memoize use.

**Thread state.** David Rowley asked for a case where Memoize is chosen on a DEFAULT_NUM_DISTINCT estimate, noting cost_memoize_rescan() guards against it. The author agreed he has none, said the patch instead widens Memoize use, and attached a new reproducer; no further replies.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Benefit is shown only on synthetic disjoint-key scenarios

fixable · level 1b, rubric item 3 · demand, planner behaviour change · `demand-1`

A common pattern is a UNION ALL of current and archive tables whose keys overlap heavily. With the patch, the planner uses hi and estimates far more groups than exist. It may then change aggregation or join plans, and the user notices only when a query that was fine before slows down after upgrade.

All evidence in message #1 comes from the author's own 18-scenario matrix and reproducers built around disjoint arms; no field report or user story is cited. The Memoize motivation changed after message #2, and message #3 offers only new author-built cases. The problem itself is plausible, but which real query shapes gain, and how often arms overlap, decides whether the change helps most users or just moves the error.

Thread: [#1 (zengxx, 2026-10-15)](https://www.postgresql.org/message-id/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com), [#2 (David Rowley, 2026-10-15)](https://www.postgresql.org/message-id/CAApHDvqjEiQ9h9DwTnwtm30cUk9Ad7R3C3vLCZj%2BdOTTz-UB-Q%40mail.gmail.com), [#3 (zengxx, 2026-10-16)](https://www.postgresql.org/message-id/tencent_AF4D43955D36DFCFCF34FE117DAC0ABEB008%40qq.com)

**Question:** Which real workloads, from users or bug reports, hit the DEFAULT_NUM_DISTINCT estimate on UNION ALL columns, and how much do their arms overlap?

Counter-check: held. The thread cites no user report, and message #1 says the ends do not infer overlap. With many overlapping arms, hi can overstate group counts by up to the number of arms, and the material shows no such case measured.

### 2. Two ndistinct ends make every future consumer choose, and nothing checks

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change, ABI / extension API · `maintenance-fit-1`

A later patch adds a planner consumer that treats ndistinct as an output cardinality and calls get_variable_numdistinct(). For UNION ALL parents it silently gets lo, the largest single member, and undercounts groups. No test or assertion fails, so the bad estimate surfaces only as a user's slow plan report.

Patch 0001 returns lo from get_variable_numdistinct() by default. Patch 0002 adds hi only through consumer-specific rules in selfuncs.c for GROUP BY, DISTINCT, Memoize and semi/anti joins. The correct end now depends on how each caller uses the number, and the proposal adds no API shape that forces a caller to state that role. Message #1 calls this split "consumer-specific safety policy", which a reviewer of an unrelated future patch is likely to miss.

Thread: [#1 (zengxx, 2026-10-15)](https://www.postgresql.org/message-id/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com)

**Question:** How does the API make a new caller pick lo or hi explicitly, so that a caller using the count as a cardinality cannot get lo by default?

Counter-check: held. Patch 0001 makes get_variable_numdistinct() return lo, and patch 0002 adds hi only through per-consumer rules. The material shows no mechanism that makes a new caller state its role, so the scenario stands as a fixable design question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Patch 0001 changes pathnodes.h, paths.h and selfuncs.h, and get_variable_numdistinct() now returns lo for flattened UNION ALL parents. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Semi/anti join selectivity, GROUP BY/DISTINCT group counts and Memoize costing change for flattened UNION ALL parents. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each consumer of ndistinct must now pick lo or hi, through per-consumer rules in selfuncs.c. |
| performance on the hot path | **yes** | The planner walks member estimates lazily; message #1 reports planning-time increases in 64-arm cases, not statistically significant. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What estimates and plans does the patch give for GROUP BY, DISTINCT and Memoize when the UNION ALL arms overlap heavily, for example a current table and an archive table?
2. How does the API make a new caller pick lo or hi explicitly, so that a caller using the count as a cardinality cannot get lo by default?
3. Which real workloads, from users or bug reports, hit the DEFAULT_NUM_DISTINCT estimate on UNION ALL columns?
4. What are the planning-time numbers for wide unions, such as the 64-arm cases, before and after the patch?
5. Are inheritance and partitioned appendrels or non-flattened UNION ALL subqueries affected in any way, and if not, why is flattened UNION ALL the right scope?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. David Rowley replied in message #2, but only on the Memoize claim. A committer who knows selfuncs.c and the appendrel planning code would have to sign off on the lo/hi policy.
- **Liability:** The owner signs for plan changes after upgrade on every query over a flattened simple UNION ALL, including UNION ALL views. The owner also signs for the rule that each future ndistinct consumer picks the correct end, for as long as that rule exists.
- **Author follow-up:** The author answered the committer's question within a day, accepted the correction and attached a new reproducer.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch adds no user-visible surface and changes estimates only for flattened simple UNION ALL. The two patches are small but change selfuncs.c and allpaths.c, and the lo/hi policy needs a planner reviewer.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Special case for one appendrel kind, with no engaged area owner. Refuted (no failure scenario): The scenario describes today's behavior for query shapes the patch leaves unchanged, so it is not a failure the patch causes. A planner committer replied within a day, and ownership belongs in the committability field, which must not raise the verdict.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P14] also concerns appendrel estimates, but it records no reason, so nothing can be carried over. [P1] was reverted because it built a cost model on multi-column ndistinct and group-size data that the planner does not have. That reason does not fit here: this proposal uses single-column member estimates and states that overlap is unknown. The thread does not point to any earlier discussion of the same idea.
- standards-compat: 0 objection(s). The proposal adds no syntax, keyword, GUC, catalog object, psql feature or dump content, so there is nothing to check for standard conflicts or pg_dump round-trips. The only visible effect is that plans change after upgrade for queries over flattened simple UNION ALL, including UNION ALL views. Estimate changes in a major release are normal, and the material shows no concrete case where the new plans are worse, so I raise no objection from this angle.
- replication-ondisk: 0 objection(s). The proposal changes only planner estimates and creates no on-disk, WAL, catalog, replication or wire-protocol state. The header changes in pathnodes.h and selfuncs.h are master-only planner internals, and the patch is not proposed for backpatching, so this angle finds nothing to object to.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. It changes internal ndistinct estimates for flattened simple UNION ALL parents, built only from member statistics that the planner already reads. Security-barrier views keep their existing behavior. It adds no SQL surface, GUC, privilege or new data path. The only thing that cannot be checked without the code is whether the member lookups keep the usual statistics privilege checks that examine_variable applies.
- maintenance-fit: 2 objection(s). The cost falls only on queries that contain flattened UNION ALL, so no hot-path objection is raised; the stress-case planning numbers in message #1 were not significant.
- already-exists: 0 objection(s). Core has no feature, setting or documented idiom that gives ndistinct for a flattened UNION ALL parent column. pg_statistic, inherited stats from ANALYZE and extended statistics all attach to real relations, not to a pulled-up subquery parent. Rewriting the query as a partitioned table or a materialized view changes the schema, so it is not an equivalent alternative. I cannot name with confidence an extension or an existing hook that gives these estimates, and the proposal adds no second mechanism alongside an existing one.
- demand: 1 objection(s). The underlying problem is a known planner weakness and the cost falls mainly on queries that use flattened UNION ALL, so demand is a minor concern here.

### Precedents retrieved

- P1: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - used in the review
- P2: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P3: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P4: [Typos in the comment for the estimate_multivariate_ndistinct()](https://commitfest.postgresql.org/patch/5740/) (cf, Withdrawn, 2025-05-07) - judged not relevant; reason summarised from its thread
- P5: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant; reason summarised from its thread
- P6: [Memoize ANTI and SEMI JOIN inner](https://commitfest.postgresql.org/patch/5636/) (cf, Withdrawn, 2025-06-30) - judged not relevant; reason summarised from its thread
- P7: [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://commitfest.postgresql.org/patch/7034/) (cf, Returned with feedback, 2026-09-10) - judged not relevant; reason summarised from its thread
- P8: [adaptive ndistinct estimator](https://commitfest.postgresql.org/patch/61/) (cf, Returned with feedback, 2015-05-01) - judged not relevant; reason summarised from its thread
- P9: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P10: [Removing useless DISTINCT clauses](https://commitfest.postgresql.org/patch/1369/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P11: [Remove useless distinct clauses](https://commitfest.postgresql.org/patch/2663/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P12: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P14: [Adjust tuples estimate for appendrel](https://commitfest.postgresql.org/patch/4711/) (cf, Returned with feedback, 2024-10-07) - used in the review
- P15: [Plan a filtering inner join as a semijoin](https://commitfest.postgresql.org/patch/7202/) (cf, Withdrawn, 2026-08-27) - judged not relevant
- P16: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant
- P17: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - judged not relevant
- P18: [Adaptive query optimization: create extented statistics using auto_explain extension](https://commitfest.postgresql.org/patch/2958/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P19: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P21: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant
- P22: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P23: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant
- P24: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 116 s; cost reported by the CLI: $1.17
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 58 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:57:56.918Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
