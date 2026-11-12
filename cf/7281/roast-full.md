# Design pre-review: Improve Hash/Merge Join estimate accuracy when all predicates are Hash/Merge clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7281](https://commitfest.postgresql.org/patch/7281/) (Needs review); authors: Ilia Evdokimov (ilidock95)
- Thread: [Improve Hash/Merge Join estimate accuracy when all predicates are Hash/Merge clauses](https://www.postgresql.org/message-id/flat/7abcff03-b38c-432f-81fa-d66abf0f061d%40tantorlabs.com), started 2025-07-10
- Messages read: 7 of 7
- Patch prose read: v4-0002-Use-exact-join-size-estimate-for-plain-inner-merg.patch, v4-0001-Account-for-NULLs-in-FK-based-join-selectivity.patch, v1-0001-Improve-row-count-estimates-for-hash-merge-joins.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the goal of more accurate merge and hash join costs is not disputed. The open points are fixable: demand-1 asks for evidence that plan choice improves, standards-compat-1 asks whether the "stable" target means back-patching, and maintenance-fit-2 notes that nobody who knows selectivity estimation has judged the tradeoff in 0001. That tradeoff, raised in message #6 and accepted in message #7, needs a committer who knows the area.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check that 0002's condition matches exactly the joins whose only join clauses are the merge or hash clauses. It also could not check how 0001 detects strict restriction clauses and combines null fractions.

## The proposal, restated

**Problem.** final_cost_mergejoin() and final_cost_hashjoin() get mergejointuples/hashjointuples from approx_tuple_count(), which treats join clauses as independent. When the clauses match a multi-column FK, this underestimates the tuples passing the join quals, so the run cost is too low (5x in the author's example). Separately, FK-based join selectivity ignores NULLs in the referencing columns and can overestimate join size.

**Proposed design.** 0002: for a JOIN_INNER join whose only join clauses are the merge/hash clauses, costing reuses path->jpath.path.rows instead of calling approx_tuple_count(). 0001: get_foreign_key_join_selectivity() scales the FK selectivity down by the table-wide null fraction of the referencing columns, using the largest per-column fraction for multi-column FKs. Columns with a strict restriction clause are skipped.

**User-visible surface.**
- Merge and hash join run costs change in EXPLAIN output, which can change plan choice.
- Join row estimates change for FK joins where referencing columns contain NULLs.

**What the author claims.**
- The joinrel estimate is more accurate and free to reuse, since it comes from FK-aware clauselist_selectivity().
- For JOIN_INNER only, path.rows equals the tuples passing the merge/hash quals; v1's two regressions came from semi and outer joins.
- FK selectivity overestimates join size by 2x when half the referencing values are NULL; 0001 removes the XXX comment's two objections.
- Taking the largest per-column null fraction is exact when FK columns are NULL together and a lower bound otherwise.
- Regression tests are fragile here because only run costs change, which costs-off EXPLAIN does not show.

**Thread state.** Xiangxin Zeng found an FK-with-NULLs case breaking v3, which led to 0001, then a case where 0001 underestimates because NULLs correlate with a filter on another column. The author accepts the tradeoff exists, and both ask for review from someone familiar with selfuncs.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-07-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No real query shown where the cost change alters the chosen plan

fixable · level 1b, rubric item 4 · demand, planner behaviour change, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

After upgrade, a user with a stable workload sees some inner merge or hash joins switch to a different plan, because their run cost rose by cpu_tuple_cost per extra tuple. The material does not show whether these switches make queries faster or slower, so users find out from changed query times in production.

Every example in the thread is synthetic and turns off enable_hashjoin and enable_nestloop, so the plan never changes (messages #3, #4, #6). The 0002 delta in message #3 is 800 on a cost of about 10,900, and no benchmark shows that the change picks a better plan. 0002 applies to all plain inner merge and hash joins, so every user takes the risk of plan changes. Message #7 accepts that 0001 can underestimate when filters correlate with NULLs, which adds to that risk.

Thread: [#3 (Ilia Evdokimov, 2026-09-09)](https://www.postgresql.org/message-id/d5c69231-5669-4d5b-a821-0b97f102a5ae%40tantorlabs.com), [#4 (Xiangxin Zeng, 2026-09-27)](https://www.postgresql.org/message-id/179051307018.1124.2392986466048370734.pgcf%40coridan.postgresql.org), [#6 (zengxx, 2026-10-14)](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7 (Ilia Evdokimov, 2026-10-19)](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com)

**Question:** Is there a real or benchmark workload, with planner settings left at their defaults, where either patch changes the chosen plan and the query time improves?

Counter-check: held. Every example in messages #3, #4 and #6 turns off enable_hashjoin and enable_nestloop, so none shows a plan change or a query time. The thread shows better cost accuracy only, and 0002 changes costs for all plain inner merge and hash joins.

### 2. Target "stable" would ship planner estimate changes in minor releases

fixable · level 1b, rubric item 6 · backpatchability, planner behaviour change · `standards-compat-1`

A site applies a minor update on a branch where 0001 was back-patched. A query with a filter correlated with FK nullness, as in message #6, has its join estimate halved. The planner then picks a nested loop or a different join order, and the site finds a slower query in production with no change of its own to explain it.

The CommitFest entry targets "stable", and nothing in the thread says whether back-patching is intended. Both patches change costs, and 0001 also changes join row estimates that feed every plan above the join. Message #7 accepts a known underestimate when NULLs correlate with a filter. Back branches normally avoid estimate changes that can flip plans without the user asking, so a committer would likely ask that the target be master only.

Thread: [#6 (zengxx, 2026-10-14)](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7 (Ilia Evdokimov, 2026-10-19)](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com)

**Question:** Is this series meant for master only, and if not, why should known plan-changing estimate shifts go into minor releases?

Counter-check: held. The CommitFest entry says "Target version: stable" and no message says whether back-patching is intended. Message #7 accepts a known underestimate, so the question about minor releases stands, and a statement of master only would answer it.

### 3. No one familiar with selectivity estimation has reviewed the FK change

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-2`

0001 is committed with its independence assumption between FK nullness and other filters. Users whose filters correlate with NULL FK values see join estimates halve, as in message #6, and get worse plans. The committer who pushed it owns these reports without an expert to judge whether the tradeoff was right.

Both participants ask for review from someone who knows selfuncs and cost estimation (messages #6, #7), and none has replied. The author agrees that 0001 swaps one assumption for another and that the scope may need to be narrower (message #7). FK-based join selectivity has been reverted before over bugs found after commit [P1], so changes in this function need an owner who knows the area.

**Precedents.**
- [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (Reverted, 2016-06-07)

Thread: [#6 (zengxx, 2026-10-14)](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7 (Ilia Evdokimov, 2026-10-19)](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com)

**Question:** Has a committer who knows get_foreign_key_join_selectivity() reviewed whether the table-wide null fraction is an acceptable tradeoff against the case in message #6?

Counter-check: held. Messages #6 and #7 both ask for review by someone who knows selfuncs, and no such reply is shown. P1 must not be used as support, because it records bugs in the first FK selectivity implementation, which do not bear on this change.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | 0002 changes merge and hash join run costs, and 0001 changes FK join row estimates, so plan choice can change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets "stable", and the thread does not say whether back-patching is intended. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0001 replaces one estimation assumption in get_foreign_key_join_selectivity() with another, and no area expert has reviewed it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No real query shown where the cost change alters the chosen plan |

## 5. Questions the author must answer before anyone reviews code

1. Is there a workload, with planner settings at their defaults, where either patch changes the chosen plan and the query time improves?
2. Is the series meant for master only, and if not, why should plan-changing estimate shifts go into minor releases?
3. Should 0001 apply only when the referencing rel has no restriction clauses, given the underestimate shown in message #6?
4. Which regression outputs change with v4, given that v1 changed several expected files and v4 touches only costsize.c?
5. With MATCH SIMPLE and partly NULL multi-column FKs, how far can the largest per-column null fraction be from the true fraction of non-matching rows?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows join selectivity estimation and costsize.c, in particular get_foreign_key_join_selectivity() and the merge and hash join cost functions.
- **Liability:** The owner signs for plan changes on every plain inner merge or hash join, and for FK join estimates that drop when NULLs correlate with a filter. Reports can arrive for as long as the estimates stay in the planner.
- **Author follow-up:** The author posted four versions and answered the reviewer's cases in the thread, with a new patch for the first one. The thread was idle for over a year between message #1 and message #2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Multi-column and nullable FK joins are common, and both patches are small and touch only costsize.c. Both participants ask for review by someone who knows selfuncs, and none has replied; review cost is set to medium because no code was read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Cost invariant in 0002 has no test that would catch a break. Refuted (no failure scenario): The scenario depends on an unnamed future patch, and a drift would be a less accurate cost estimate, which approx_tuple_count() already is. v1's semi and outer join cases did show up, in changed regression output and in the author's table in message #1.
- `maintenance-fit-3` Target version "stable" for changes that alter plan choice. Refuted (not specific): Duplicate of standards-compat-1, which states the same back-patching concern with a more concrete scenario.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent is the same idea, and none has a recorded reason that applies here. [P1] and [P3] built the FK-based join estimate that 0001 adjusts, but they ended over bugs, planner slowdown and a stalled revision. [P2] used extended statistics at the wrong abstraction level, and [P18] has no recorded reason. Inside the thread, the only earlier decision is the XXX comment in get_foreign_key_join_selectivity(). The 0001 commit message says it handles both of that comment's reasons, and message #6 raises a different issue, the correlation between nullness and filters, which belongs to the design angle.
- standards-compat: 1 objection(s). The proposal adds no syntax, GUC, catalog object or psql change, so it has no pg_dump, standard or client surface. The only objection from this angle concerns the "stable" target.
- replication-ondisk: 0 objection(s). Both patches change only planner cost and row estimates in costsize.c. They add no on-disk format, WAL record, catalog column, wire protocol message or extension-visible struct, so pg_upgrade, standbys, subscribers and mixed-version clusters are not affected. The CommitFest target is "stable", which would mean plan changes in a minor release, but that is a planner-behaviour question for another angle and the material does not say back-patching is intended.
- security: 0 objection(s). The proposal changes planner cost and row estimates only: merge/hash join run costs and FK join selectivity from existing pg_statistic null fractions. It adds no privilege, object, function, GUC, view or connection path, and exposes no statistics beyond what EXPLAIN already shows, so the security angle has no objection.
- maintenance-fit: 3 objection(s)
- already-exists: 0 objection(s). Postgres has no existing feature, option or extension that corrects the merge/hash join run cost or the NULL-blind FK selectivity, and both patches extend existing estimator code rather than adding a second mechanism. Whether 0001's independence assumption is acceptable is a design question for the selfuncs angle, outside this one.
- demand: 1 objection(s). Multi-column and nullable FK joins are common, so the underlying estimation problem is plausible. The only demand gap is evidence that the cost change helps real plan choices.

### Precedents retrieved

- P1: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - used in the review
- P2: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P3: [improving join estimates using FK](https://commitfest.postgresql.org/patch/210/) (cf, Returned with feedback, 2016-01-31) - used in the review; reason summarised from its thread
- P4: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P6: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P7: [PoC: improving join estimates using sampling](https://commitfest.postgresql.org/patch/3211/) (cf, Returned with feedback, 2022-03-22) - judged not relevant; reason summarised from its thread
- P8: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant; reason summarised from its thread
- P9: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant
- P11: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P12: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P13: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P14: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P15: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P16: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P17: [Typos in the comment for the estimate_multivariate_ndistinct()](https://commitfest.postgresql.org/patch/5740/) (cf, Withdrawn, 2025-05-07) - judged not relevant
- P18: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - used in the review
- P19: [Join Correlation Name](https://commitfest.postgresql.org/patch/2322/) (cf, Withdrawn, 2019-11-01) - judged not relevant
- P20: [Better estimate for filter cost](https://commitfest.postgresql.org/patch/1296/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P21: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [Range Merge Join](https://commitfest.postgresql.org/patch/1106/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P23: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P24: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 123 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:37:01.773Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
