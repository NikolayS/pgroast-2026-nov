# Design pre-review: Apply extended statistics to join clause during parameterized path costing

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7161](https://commitfest.postgresql.org/patch/7161/) (Needs review); authors: Ilia Evdokimov (ilidock95)
- Thread: [Apply extended statistics to join clause during parameterized path costing](https://www.postgresql.org/message-id/flat/7b1bd383-fa74-4c0e-a75a-52ff9c753750%40tantorlabs.com), started 2026-08-17
- Messages read: 3 of 3
- Patch prose read: v1-0001-Use-extended-statistics-for-join-clauses-during-p.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in message #2, but the design step is disputed. prior-attempts-1 survives: the patch changes the parameterized scan estimate while the join estimate stays at 98, and message #3 defers the join-level fix without saying why the scan-only step is safe alone. demand-1 adds that the only example forces a nested loop and runs slower after the change. The thread needs agreement on scope before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. It could not be checked how non-equality operators or expressions on the other side are handled, whether the new path keeps the existing column-privilege checks, or whether it can apply a dependency twice alongside other estimates.

## The proposal, restated

**Problem.** Extended statistics apply only when every clause references one relation, so a parameterized inner scan of a nested loop falls back to per-column independence. With correlated join columns this underestimates rows per probe, and the planner can pick a poor inner access path. The example shows 2 estimated rows per probe against 200 actual.

**Proposed design.** dependency_is_compatible_clause() also accepts clauses of the form relid.a = otherrel.b, with exactly two varnos, and treats the other relation's Var as a pseudoconstant. When a clause list spans relations but is estimated for a specific varRelid, the planner looks up that relation's dependencies statistics and applies them through dependencies_clauselist_selectivity. The degree logic is unchanged, and only the dependencies kind is used.

**User-visible surface.**
- Row estimates and costs of parameterized inner scans change when dependencies statistics exist on correlated join columns, which can change plan choice.

**What the author claims.**
- With dependencies statistics, the per-probe estimate in the example moves from 2 rows to 200, matching the actual count.
- The nested loop cost changes from 579.23 to 4474.19, because inner path costing uses inner_path->rows.
- ndistinct and MCV support would need much more code and should be separate follow-ups.
- Join-level estimation in calc_joinrel_size_estimate is a separate, larger change, and related work is linked.

**Thread state.** Tomas Vondra supports using per-table extended statistics for joins, asks why ndistinct and MCV are not used, and objects that the scan estimate now disagrees with the join estimate (50 * 200 != 98). The author replied, proposing dependencies first and the join-level fix later; nobody has replied since, and the scope question is open.

## 2. Prior attempts

### Improve join selectivity estimation using extended statistics (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2959/](https://commitfest.postgresql.org/patch/2959/)
- **Thread:** [https://www.postgresql.org/message-id/flat/71d67391-16a9-3e5e-b5e4-8f7fd32cc1b2@postgrespro.ru](https://www.postgresql.org/message-id/flat/71d67391-16a9-3e5e-b5e4-8f7fd32cc1b2@postgrespro.ru)
- **Date:** decided 2021-10-01; started 2021-01-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tomas Vondra, Ibrar Ahmed (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tomas Vondra found the design at the wrong level of abstraction. Clausesel.c handled extended statistics directly, dependencies were applied clause by clause in a fixed order and only on the inner relation, and he proposed handling all join clauses for a pair of relations at once in extended_stats.c. The author agreed it could be done better but posted no new version. In July the patch no longer compiled, and it still did not one commitfest later, so it was returned with feedback.
- **Reason (quote):** "The patch includes some *very* interesting ideas, but I think it's does them too late and at the wrong level of abstraction." ([message](https://www.postgresql.org/message-id/2a7a4c58-d9b4-5529-945a-0dcba6f6ef22%40enterprisedb.com))
- **What is different this time:** This patch narrows the scope to parameterized inner-path costing and leaves calc_joinrel_size_estimate alone. P2 also tried join selectivity and conditional probabilities with relation filters.

### Using multicolumn statistics for more precise join selectivity estimatation (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2386/](https://commitfest.postgresql.org/patch/2386/)
- **Thread:** [https://www.postgresql.org/message-id/flat/196f1e1a-5464-ed07-ab3c-0c9920564af7@postgrespro.ru](https://www.postgresql.org/message-id/flat/196f1e1a-5464-ed07-ab3c-0c9920564af7@postgrespro.ru)
- **Date:** decided 2021-10-01; started 2019-10-14
- **Outcome:** Returned with feedback (cf)
- **Who objected:** vignesh C, Yugo NAGATA, Tomas Vondra (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The last visible message (July 2021) reports that the patch no longer compiled on Windows against changed ModifyTableState fields. The same message says Yugo Nagata's open review comments still needed addressing, and the entry was moved to Waiting on Author. Those comments included duplicate statistics suggestions causing a unique-constraint error, redundant suggestions, and a request to explain why each statistic is suggested. Earlier, Tomas Vondra had suggested moving the advisor into a separate extension rather than auto_explain, collecting data across many runs, and not creating statistics automatically. The visible messages show no new version after that, and the entry was returned with feedback.
- **Reason (quote):** "Also Yugo Nagata's comments need to be addressed, I'm changing the status to "Waiting for Author"." ([message](https://www.postgresql.org/message-id/CALDaNm1THv0os7C%3DwF44fGBNVoTBf%2B7xWRAuc87%2Bto%2BNMPUdxg%40mail.gmail.com))
- **What is different this time:** P6 used functional dependencies for join selectivity and bundled an auto_explain statistics advisor. Its recorded reasons concern build failures and the advisor part, which do not apply here.

### estimating joins using extended statistics (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3055/](https://commitfest.postgresql.org/patch/3055/)
- **Thread:** [https://www.postgresql.org/message-id/flat/c8c0ff31-3a8a-7562-bbd3-78b2ec65f16c@enterprisedb.com](https://www.postgresql.org/message-id/flat/c8c0ff31-3a8a-7562-bbd3-78b2ec65f16c@enterprisedb.com)
- **Date:** decided 2022-02-07; started 2021-03-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund, Julien Rouhaud (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** No reviewer objected to the design in the visible messages. The last messages report that the 2022-01-01 version fails cfbot with uninitialized-variable warnings in mcv_combine_simple() and no longer applies to master, and no new version appears before the entry was returned. The author still listed open items, such as no support for outer or inequality joins and possibly wrong ndistinct and nullfract assumptions for column subsets.
- **Reason (quote):** "Also the patch doesn't apply anymore:" ([message](https://www.postgresql.org/message-id/20220119101809.wzrjvh4jsnoakgny%40jrouhaud))
- **What is different this time:** P1 matched multi-column MCV lists of both relations at join level, like eqjoinsel_inner. This patch uses only dependencies on one relation, at scan level. P1 stopped because the patch no longer applied; no design objection is recorded.


## 3. Objections a reviewer is likely to raise

### 1. Repeats the per-relation clausesel.c approach that an earlier attempt was told to restructure

fixable · level 2, rubric item 12 · prior rejection, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user adds dependencies statistics on big(a, b). The parameterized inner scan is now costed for 200 rows per probe, but the join estimate stays at 98. Upper joins and aggregates plan on 98 rows while the nested loop is costed for 10000. The user sees plans change after CREATE STATISTICS, and EXPLAIN shows row counts that do not multiply out.

[P2] also applied dependencies only on the inner relation inside clausesel.c. It was returned with "too late and at the wrong level of abstraction", with a suggestion to handle all join clauses for a relation pair at once. Message #2 raises the same kind of concern in a new form: the scan estimate moves and the join estimate does not. Message #3 defers the join-level fix but does not say why the scan-only step is safe on its own, and nobody has replied since.

**Precedents.**
- [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (Returned with feedback, 2021-10-01); thread: [link](https://www.postgresql.org/message-id/flat/71d67391-16a9-3e5e-b5e4-8f7fd32cc1b2@postgrespro.ru)

Thread: [#2 (Tomas Vondra, 2026-08-21)](https://www.postgresql.org/message-id/e651725f-4bf0-4fa6-8c5d-db27d96a2e31%40vondra.me), [#3 (Ilia Evdokimov, 2026-09-15)](https://www.postgresql.org/message-id/24bf4ee2-3904-47d0-8181-96d23f3fa201%40tantorlabs.com)

**Question:** Why is changing only the parameterized scan estimate, with join rows unchanged, a safe first step and not a structure that the join-level fix will have to replace?

Counter-check: held. Message #2 raises the scan and join mismatch (50 * 200 != 98), and message #3 defers it without saying why the scan-only step is safe. [P2] concerned join selectivity, so it supports the placement concern only loosely.

### 2. No workload shown where the corrected estimate gives a better plan

fixable · level 1b, rubric item 2 · desirability, planner behaviour change · `demand-1`

A user with dependencies statistics on join columns upgrades, and the parameterized inner scans in their nested loops get new estimates and new access paths. In the only example given, the plan moves from an index scan to a bitmap heap scan, and execution time goes from about 2.7 ms to 14.4 ms. The user sees the slowdown after the upgrade and has no setting to get the old plan back.

The example forces a nested loop with enable_mergejoin and enable_hashjoin off, so it shows a corrected estimate, not a better plan. Message #1 does not discuss that its new plan runs about 5 times slower. The thread has no field report and no -performance or -bugs case where this misestimate led to a bad plan. The broader demand for using extended statistics in join estimation is real, as message #2 agrees, but this patch changes only the scan estimate and only for the dependencies kind.

Thread: [#1 (Ilia Evdokimov, 2026-08-17)](https://www.postgresql.org/message-id/7b1bd383-fa74-4c0e-a75a-52ff9c753750%40tantorlabs.com), [#2 (Tomas Vondra, 2026-08-21)](https://www.postgresql.org/message-id/e651725f-4bf0-4fa6-8c5d-db27d96a2e31%40vondra.me)

**Question:** Which real query, without forced enable_* settings, gets a faster plan from this change, and how often does the plan get slower instead?

Counter-check: held. The only example forces a nested loop and runs slower after the change, and the thread does not discuss it. Message #2 supports the general direction, but the material shows no query that gets a better plan from this step.

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
| planner behaviour change | **yes** | Row estimates and costs of parameterized inner scans change when dependencies statistics exist, which can change the chosen access path. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry lists target version "stable", and the patch changes plans for existing statistics objects. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a scan-level path in clausesel.c that later join-level work in eqjoinsel() or calc_joinrel_size_estimate must stay consistent with. |
| performance on the hot path | **yes** | Planning now looks up dependencies statistics for cross-relation clause lists; the example shows planning time of 0.110 ms before and 0.648 ms after, over one run. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is changing only the parameterized scan estimate, with join rows unchanged at 98, a safe first step and not a structure the join-level fix must replace?
2. How do plans above the join behave when the inner scan is costed for 200 rows per probe but the join reports 98 rows?
3. Which query, without forced enable_* settings, gets a faster plan from this change, and how often does the plan get slower?
4. How does the example perform over repeated warm-cache runs, given that execution time went from 2.690 ms to 14.428 ms in the single run shown?
5. How much planning time does the new lookup add for joins on tables that have dependencies statistics, measured over many runs?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tomas Vondra reviewed in message #2 and supports the general direction. The owner must know both extended statistics and join selectivity estimation.
- **Liability:** The owner signs for plan changes on every site that has dependencies statistics on join columns, and for keeping the scan-level path consistent with later join-level estimation work.
- **Author follow-up:** The author answered the review in message #3 and proposes follow-up patches for ndistinct, MCV and join-level estimation.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is 4.3 KB in two planner files and helps only users with dependencies statistics on correlated join columns in nested loops. The scope question from message #2 is open, and the scan and join estimates disagree until it is settled.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Existing dependencies statistics silently change nested loop plans after upgrade. Refuted (not specific): Every planner estimate improvement changes plans for existing statistics at a major upgrade, and earlier extended statistics work did the same with no opt-out. The thread does not ask for a backpatch, and the missing documentation is outside levels 1-2.
- `maintenance-fit-1` Second estimation path for join clauses that join-level work must reconcile. Refuted (not specific): Duplicate of prior-attempts-1: it rests on the same scan and join mismatch and the same [P2] placement point. Its added scenario of a later mismatch that ships unnoticed is speculative.
- `maintenance-fit-2` Plan changes for existing statistics users, with a slower example. Refuted (not specific): Duplicate of demand-1, which makes the same point about the slower example and adds that the forced nested loop shows no better plan.

### Review angles

- prior-attempts: 1 objection(s). Message #1 links related ongoing work [0], and message #3 links a join-level discussion. The thread gives no outcome for either. P4, P9, P16 and P18 share the area, but their recorded reasons do not apply.
- standards-compat: 1 objection(s). From the standards angle the patch adds no syntax, GUC, catalog entry or dump-visible object. Only the plan change for existing statistics users remains.
- replication-ondisk: 0 objection(s). The proposal changes only planner selectivity estimation in clausesel.c and dependencies.c. It adds no on-disk format, WAL record, catalog column, statistics format, or protocol change, and reads existing pg_statistic_ext_data dependencies as they are. Exported function signatures in dependencies.c could matter for extensions, but the material does not show any signature change.
- security: 0 objection(s). The proposal has no privilege or exposure aspect that I can tie to a failure scenario. It changes only row estimates for parameterized inner scans, using dependencies statistics that the planner already reads for single-relation clauses. Dependencies store only a degree per column pair and no data values, and the patch does not evaluate operators against stored values, so leaky-operator and row-level security concerns do not apply as they would for MCV lists. Whether the new code path keeps the existing column-privilege checks in extended statistics cannot be checked without the code. Planning cost and plan quality belong to other angles.
- maintenance-fit: 2 objection(s). Planning overhead seems to fall mostly on tables that have dependencies statistics, so I raise no hot-path objection, but the proposal does not yet measure it.
- already-exists: 0 objection(s). Core Postgres has no way to apply a relation's dependencies statistics to the per-probe estimate of a parameterized inner scan, and I know of no extension that does this. Hint extensions can fix join row counts but not inner path rows. The overlap with join-level work ([P2], [P1], Tomas's eqjoinsel point in message #2) is a design question for other angles.
- demand: 1 objection(s). Demand for using extended statistics in join estimation is real, and message #2 says so. The objection is limited to missing evidence that this narrow change helps a real plan.

### Precedents retrieved

- P1: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P2: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P3: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P4: [Extended statistics / estimate Var op Var clauses](https://commitfest.postgresql.org/patch/2831/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P5: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P6: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P7: [Showing applied extended statistics in explain Part2](https://commitfest.postgresql.org/patch/5077/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant; reason summarised from its thread
- P9: [Add support for (Var op Var) clause in extended MCV statistics](https://commitfest.postgresql.org/patch/5180/) (cf, Returned with feedback, 2024-09-30) - used in the review
- P10: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P11: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P13: [OpenSSL 3.0.0 compatibility](https://commitfest.postgresql.org/patch/2810/) (cf, Withdrawn, 2021-03-10) - judged not relevant
- P14: [Join Correlation Name](https://commitfest.postgresql.org/patch/2322/) (cf, Withdrawn, 2019-11-01) - judged not relevant
- P15: [multivariate statistics](https://commitfest.postgresql.org/patch/120/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P16: [PoC: improving join estimates using sampling](https://commitfest.postgresql.org/patch/3211/) (cf, Returned with feedback, 2022-03-22) - used in the review
- P17: [multivariate statistics](https://commitfest.postgresql.org/patch/450/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P18: [improving join estimates using FK](https://commitfest.postgresql.org/patch/210/) (cf, Returned with feedback, 2016-01-31) - used in the review
- P19: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P20: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P21: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P22: [bitmap cost should account for correlated indexes](https://commitfest.postgresql.org/patch/2310/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P23: [multivariate statistics](https://commitfest.postgresql.org/patch/58/) (cf, Rejected, 2015-01-16) - judged not relevant
- P24: [Pulling up direct-correlated ANY_SUBLINK](https://commitfest.postgresql.org/patch/3941/) (cf, Returned with feedback, 2023-01-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 111 s; cost reported by the CLI: $1.47
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 7 of 24 kept
- Tone check: 74 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:27.395Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
