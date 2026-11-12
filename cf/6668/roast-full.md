# Design pre-review: Implement missing join selectivity estimation for range types

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6668](https://commitfest.postgresql.org/patch/6668/) (Needs review); authors: Haibo Yan (billyean), Maxime Schoemans (mschoema), Mahmoud SAKR (mahmsakr), Diogo Repas (diogo.repas), Zhicheng Luo (zcsysu.163.com)
- Thread: [Implement missing join selectivity estimation for range types](https://www.postgresql.org/message-id/flat/CAB4o4asMq3k6HN9WfDsssQ5DDVfAziB4TpiFJ8RBJgZTVuwC7g%40mail.gmail.com), started 2022-06-30
- Messages read: 32 of 32
- Patch prose read: v1-0001-Join-Selectivity-Estimation-for-Range-types.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal or the design of the estimator. demand-1 shows the thread has no user report, but the cost falls only on queries that join on range columns, so it is a request for evidence. maintenance-fit-2 is decisive: no committer has reviewed the v5 to v9 rework, and the January 2024 ASan crash is not shown to be gone.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the ASan crash from January 2024, the bounds handling in the merge walk and the coverage of the security check in every estimator path could not be judged.

## The proposal, restated

**Problem.** For joins on range columns such as t1.r << t2.r, the planner uses a fixed default selectivity of 0.005 and ignores the bounds histograms that ANALYZE already collects. Bad estimates can lead to poor join order and join methods in queries that join on range columns.

**Proposed design.** rangejoinsel uses the lower and upper bounds histograms of both sides, merged in one walk, to estimate <<, >> and && (v5 onward drops &<, &>, @>, <@). A second patch applies the same outer-bounds estimator to multirange and mixed range/multirange joins. A third patch moves the helper functions duplicated in the two selfuncs files into a backend-private rangetypes_selfuncs.h.

**User-visible surface.**
- New join selectivity functions for range operators <<, >>, && in pg_proc.dat, set as oprjoin in pg_operator.dat.
- Same for multirange and mixed range/multirange operators <<, >>, &&.
- Changed row estimates and plans for range and multirange joins.
- Estimators fall back to the default selectivity when the comparator or subtype_diff function fails statistic_proc_security_check().

**What the author claims.**
- Estimation error is about 0.5% for the covered operators, based on the authors' experiments.
- The paper's algorithm beats the estimators in Oracle, SQL Server, MySQL and Postgres.
- Estimates stay accurate on data with many duplicate ranges, even without MCV statistics for ranges.
- For multiranges, && can overestimate on sparse values, but this matches existing restriction selectivity and beats the fixed default.
- v9 passes the full regression suite, and new privileges.sql tests fail without the security checks.

**Thread state.** After a 2023 review by Tomas Vondra, Tom Lane and others, Haibo Yan revived the patch in 2026 and reviewed v6 to v8 with Maxime Schoemans. Bryan Green found a missing statistic_proc_security_check(), fixed in v9, and no committer has replied since; CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2022-06-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No user report shows range or multirange joins being misestimated

fixable · level 1b, rubric item 3 · demand, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer takes on a new histogram merge-walk estimator for range and multirange joins, plus a new privilege-check path, and owns its bugs and plan changes. If few production queries join on <<, >> or && between range columns, the plan gains reach few users while the ASan history and security surface stay with the project.

The patch began as a course project (message #7), and Tomas Vondra said he did not remember ever seeing an inequality join on a range column (message #6). The demand signals are reviewers calling it useful (message #15) and synthetic tables built to show the effect (messages #7, #20). No -general or -bugs report, production plan or user workload appears in the thread. Tom Lane's point (message #13) that scalar priority is no reason to reject is a different question from whether range joins have users.

Thread: [#6 (Tomas Vondra, 2023-01-21)](https://www.postgresql.org/message-id/d46e1950-9228-3fe2-02b6-72ab7aaafb3a%40enterprisedb.com), [#7 (Mahmoud Sakr, 2023-01-25)](https://www.postgresql.org/message-id/CAB4o4asvPN%3DNT7KvS9zVQjZbdsiRW5t8aQctEkW7mxc4hbBxVQ%40mail.gmail.com), [#13 (Tom Lane, 2023-11-14)](https://www.postgresql.org/message-id/2516071.1699991181%40sss.pgh.pa.us), [#15 (Alena Rybakina, 2023-11-30)](https://www.postgresql.org/message-id/5fc8d7f6-9cd5-4da3-8a94-1d06831441fe%40yandex.ru), [#20 (Haibo Yan, 2026-04-06)](https://www.postgresql.org/message-id/CABXr29HUT2m7YPpB4gOZHphYmwpwuEWFZ_p6oeyouhWw8Zc1Tw%40mail.gmail.com)

**Question:** Which real workloads, such as temporal or scheduling schemas, join on range <<, >> or && today, and what bad plans do they get from the 0.005 default?

Counter-check: held. The thread has no user report or production plan, and message #6 says range joins are rare. The cost falls only on queries that join on range columns, so this supports asking for demand evidence and does not support a rejection.

### 2. No committer has engaged with the 2026 series

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years"), planner behaviour change, security / privileges · `maintenance-fit-2`

A committer pushes v9 late in a cycle. A buildfarm member later reproduces the ASan out-of-bounds read reported in messages #16 and #18, or a user reports a plan regression on range joins. The committer must then fix an estimator that no other committer has reviewed since 2023.

The 2023 committer comments (messages #2-#6, #13) were about v1-v3. The v5-v9 rework was reviewed only by co-authors and non-committers (messages #21-#32), and message #28 says a committer is still needed. The original code came from a course project (message #7), and message #17 says the author could not confirm the CFbot crash was fixed. The CF entry has crossed three CommitFests and now needs a rebase.

Thread: [#7 (Mahmoud Sakr, 2023-01-25)](https://www.postgresql.org/message-id/CAB4o4asvPN%3DNT7KvS9zVQjZbdsiRW5t8aQctEkW7mxc4hbBxVQ%40mail.gmail.com), [#13 (Tom Lane, 2023-11-14)](https://www.postgresql.org/message-id/2516071.1699991181%40sss.pgh.pa.us), [#16 (vignesh C, 2024-01-05)](https://www.postgresql.org/message-id/CALDaNm29NMhpPvoM49qqfYzVcRJq0CPTd-xSUcTF7RZQO2TASQ%40mail.gmail.com), [#17 (Schoemans Maxime, 2024-01-05)](https://www.postgresql.org/message-id/08b2dea8-6dca-06e4-a53d-3e6334e7645b%40ulb.be), [#18 (vignesh C, 2024-01-17)](https://www.postgresql.org/message-id/CALDaNm3A4j7%3DrmY128Eb3v8zi%3DhHAnGizUcbSX5ekZKHy-wmMA%40mail.gmail.com), [#28 (Haibo Yan, 2026-04-23)](https://www.postgresql.org/message-id/CABXr29Edby6Cbbaps_OQh-%2BkvKx%2BXFNe26jc%3DsBxWhjaYO9raQ%40mail.gmail.com), [#31 (Bryan Green, 2026-09-18)](https://www.postgresql.org/message-id/6929b807-32e9-4e80-b3d3-1f7823880d60%40gmail.com), [#32 (Haibo Yan, 2026-09-18)](https://www.postgresql.org/message-id/CABXr29GKcW58pJ%2BmqZNXjAjLEmGdsDY%2BH-5bmrkgiDywjWnBJQ%40mail.gmail.com)

**Question:** Has a committer familiar with selfuncs reviewed the v9 design, and is the 2024 ASan failure shown to be gone on CFbot?

Counter-check: held. The thread shows no committer reply after message #13, and the v5 to v9 rework had a security gap found only in September 2026. This belongs in the committability field and must not raise the verdict on the design.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds join selectivity functions to pg_proc.dat and sets them as oprjoin in pg_operator.dat. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Row estimates and plans change for joins on range and multirange <<, >> and &&. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The estimators run comparator and subtype_diff functions over statistics, guarded by statistic_proc_security_check() since v9. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two new histogram-based join estimators and a shared private header stay in the planner's selectivity code. |
| performance on the hot path | **yes** | The merge walk runs at plan time for range join clauses; its cost at high statistics targets is not shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the ASan out-of-bounds read in calc_hist_join_selectivity, reported in messages #16 and #18, shown to be gone in v9 on CFbot?
2. Which real workloads join on range <<, >> or && today, and what bad plans do they get from the 0.005 default?
3. What is the estimation error of v9 for <<, >> and && only, including data with many duplicate ranges?
4. How much planning time does the merge walk add for a range join at a statistics target of 10000?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the selectivity estimation code in selfuncs. Tomas Vondra and Tom Lane commented on v1 to v3 in 2023, and no committer has replied on v5 to v9.
- **Liability:** The owner signs for plan changes on range and multirange joins, the correctness of the histogram merge walk, and the privilege check path, for as long as the estimators exist.
- **Author follow-up:** Haibo Yan revived the patch in 2026 and posted v9 to fix the security check found in review. The entry has needed a rebase since 2026-11-05.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The goal is valid and costs fall only on range joins, but message #6 says such joins are rare and no committer has reviewed v5 to v9. Review cost is judged from the touched files; the code was not read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Merge-walk estimator is tied to range types, not shaped for scalar reuse. Refuted (no failure scenario): The failure needs a scalar patch that does not exist yet, and that patch could move the walk to a shared place. Postgres already keeps range histogram code apart from scalar code, because range bounds use RangeBound comparison and scalars also need MCV handling (messages #4, #7).

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposes this idea; [P1], [P8], [P9] and [P6] cover other join estimation methods, and their reasons do not apply. Message #29 points to this patch's own earlier CommitFest entry, which stalled on the CFbot ASan abort (messages #16 to #18), a code question outside this angle.
- standards-compat: 0 objection(s). The patch adds no syntax, keyword, GUC, psql command or dumpable object; it only changes oprjoin for built-in range and multirange operators. These changes reach users only through a major-version catalog update, and plan changes after upgrade are the expected effect of a better estimate. The && overestimate for sparse multiranges was settled in messages #23-#24. Not checked without the code: whether the oprjoin entries that are replaced are referenced anywhere outside these operators.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state beyond new built-in pg_proc entries and oprjoin settings in pg_operator.dat, which only need the usual catversion bump. It changes no on-disk format, WAL record, statistics layout, wire protocol or extension ABI, and the new rangetypes_selfuncs.h is backend-private.
- security: 0 objection(s). The only privilege question is the missing statistic_proc_security_check() before support functions run over another role's bounds histogram (message #31). v9 adds the check for both sides of both estimators, falls back to the default selectivity, and adds privileges.sql tests (message #32). The merge walk is linear in histogram size, so it gives no new denial-of-service path. The January 2024 ASan out-of-bounds read (messages #16, #18) is a code-level question this angle cannot judge without the code.
- maintenance-fit: 2 objection(s). Hot-path cost is limited to range and multirange join clauses, so non-users pay nothing. The multirange && approximation and the duplication concern were settled in messages #23-#25.
- already-exists: 0 objection(s). Postgres has no existing way to get these estimates. The range && operator uses a fixed default today, and rewriting r1 << r2 as upper(r1) < lower(r2) only reaches scalarltjoinsel, which is also a constant. No extension that does this is known to me, and estimators for built-in range operators belong next to the existing restriction estimators in rangetypes_selfuncs.c. Patch 3 removes duplicated helpers rather than adding a second mechanism.
- demand: 1 objection(s). Costs fall mostly on queries that use these operators, so this is a not now, show demand point and not grounds for rejection.

### Precedents retrieved

- P1: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P2: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P4: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P5: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P6: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - used in the review
- P7: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant; reason summarised from its thread
- P8: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P9: [PoC: improving join estimates using sampling](https://commitfest.postgresql.org/patch/3211/) (cf, Returned with feedback, 2022-03-22) - used in the review; reason summarised from its thread
- P10: [improving join estimates using FK](https://commitfest.postgresql.org/patch/210/) (cf, Returned with feedback, 2016-01-31) - judged not relevant; reason summarised from its thread
- P11: [Range Merge Join](https://commitfest.postgresql.org/patch/1106/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P12: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P13: [Range Merge Join](https://commitfest.postgresql.org/patch/1449/) (cf, Returned with feedback, 2018-03-28) - judged not relevant
- P14: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P15: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P16: [Range phrase operator in tsquery](https://commitfest.postgresql.org/patch/1624/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P17: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant
- P18: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P19: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant
- P21: [Batch processing of RI checks](https://commitfest.postgresql.org/patch/2581/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P22: [Pass down Limit bounds \[take-2\]](https://commitfest.postgresql.org/patch/866/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P23: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P24: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 148 s; cost reported by the CLI: $3.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 61 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:59:11.389Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
