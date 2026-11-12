# Design pre-review: Apply extended statistics to join clause during parameterized path costing

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7161](https://commitfest.postgresql.org/patch/7161/); [thread](https://www.postgresql.org/message-id/flat/7b1bd383-fa74-4c0e-a75a-52ff9c753750%40tantorlabs.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in message #2, but the design step is disputed. prior-attempts-1 survives: the patch changes the parameterized scan estimate while the join estimate stays at 98, and message #3 defers the join-level fix without saying why the scan-only step is safe alone. demand-1 adds that the only example forces a nested loop and runs slower after the change. The thread needs agreement on scope before code review.

Not checked: The code was not read. It could not be checked how non-equality operators or expressions on the other side are handled, whether the new path keeps the existing column-privilege checks, or whether it can apply a dependency twice alongside other estimates.

## Prior attempts

- [Improve join selectivity estimation using extended statistics](https://www.postgresql.org/message-id/flat/71d67391-16a9-3e5e-b5e4-8f7fd32cc1b2@postgrespro.ru) (similar idea; Returned with feedback, 2021-10-01). "The patch includes some *very* interesting ideas, but I think it's does them too late and at the wrong level of abstraction." ([message](https://www.postgresql.org/message-id/2a7a4c58-d9b4-5529-945a-0dcba6f6ef22%40enterprisedb.com)) Different now: This patch narrows the scope to parameterized inner-path costing and leaves calc_joinrel_size_estimate alone. P2 also tried join selectivity and conditional probabilities with relation filters.
- [Using multicolumn statistics for more precise join selectivity estimatation](https://www.postgresql.org/message-id/flat/196f1e1a-5464-ed07-ab3c-0c9920564af7@postgrespro.ru) (similar idea; Returned with feedback, 2021-10-01). "Also Yugo Nagata's comments need to be addressed, I'm changing the status to "Waiting for Author"." ([message](https://www.postgresql.org/message-id/CALDaNm1THv0os7C%3DwF44fGBNVoTBf%2B7xWRAuc87%2Bto%2BNMPUdxg%40mail.gmail.com)) Different now: P6 used functional dependencies for join selectivity and bundled an auto_explain statistics advisor. Its recorded reasons concern build failures and the advisor part, which do not apply here.
- [estimating joins using extended statistics](https://www.postgresql.org/message-id/flat/c8c0ff31-3a8a-7562-bbd3-78b2ec65f16c@enterprisedb.com) (similar idea; Returned with feedback, 2022-02-07). "Also the patch doesn't apply anymore:" ([message](https://www.postgresql.org/message-id/20220119101809.wzrjvh4jsnoakgny%40jrouhaud)) Different now: P1 matched multi-column MCV lists of both relations at join level, like eqjoinsel_inner. This patch uses only dependencies on one relation, at scan level. P1 stopped because the patch no longer applied; no design objection is recorded.

## Objections a reviewer is likely to raise

### 1. Repeats the per-relation clausesel.c approach that an earlier attempt was told to restructure

A user adds dependencies statistics on big(a, b). The parameterized inner scan is now costed for 200 rows per probe, but the join estimate stays at 98. Upper joins and aggregates plan on 98 rows while the nested loop is costed for 10000. The user sees plans change after CREATE STATISTICS, and EXPLAIN shows row counts that do not multiply out.

[P2] also applied dependencies only on the inner relation inside clausesel.c. It was returned with "too late and at the wrong level of abstraction", with a suggestion to handle all join clauses for a relation pair at once. Message #2 raises the same kind of concern in a new form: the scan estimate moves and the join estimate does not. Message #3 defers the join-level fix but does not say why the scan-only step is safe on its own, and nobody has replied since.

Precedent: [Improve join selectivity estimation using extended statistics](https://www.postgresql.org/message-id/flat/71d67391-16a9-3e5e-b5e4-8f7fd32cc1b2@postgrespro.ru) (Returned with feedback, 2021-10-01)

**Question:** Why is changing only the parameterized scan estimate, with join rows unchanged, a safe first step and not a structure that the join-level fix will have to replace?

<sub>fixable, level 2, prior rejection; messages [#2](https://www.postgresql.org/message-id/e651725f-4bf0-4fa6-8c5d-db27d96a2e31%40vondra.me), [#3](https://www.postgresql.org/message-id/24bf4ee2-3904-47d0-8181-96d23f3fa201%40tantorlabs.com); `prior-attempts-1`</sub>

### 2. No workload shown where the corrected estimate gives a better plan

A user with dependencies statistics on join columns upgrades, and the parameterized inner scans in their nested loops get new estimates and new access paths. In the only example given, the plan moves from an index scan to a bitmap heap scan, and execution time goes from about 2.7 ms to 14.4 ms. The user sees the slowdown after the upgrade and has no setting to get the old plan back.

The example forces a nested loop with enable_mergejoin and enable_hashjoin off, so it shows a corrected estimate, not a better plan. Message #1 does not discuss that its new plan runs about 5 times slower. The thread has no field report and no -performance or -bugs case where this misestimate led to a bad plan. The broader demand for using extended statistics in join estimation is real, as message #2 agrees, but this patch changes only the scan estimate and only for the dependencies kind.

**Question:** Which real query, without forced enable_* settings, gets a faster plan from this change, and how often does the plan get slower instead?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/7b1bd383-fa74-4c0e-a75a-52ff9c753750%40tantorlabs.com), [#2](https://www.postgresql.org/message-id/e651725f-4bf0-4fa6-8c5d-db27d96a2e31%40vondra.me); `demand-1`</sub>

## Questions for the author

1. Why is changing only the parameterized scan estimate, with join rows unchanged at 98, a safe first step and not a structure the join-level fix must replace?
2. How do plans above the join behave when the inner scan is costed for 200 rows per probe but the join reports 98 rows?
3. Which query, without forced enable_* settings, gets a faster plan from this change, and how often does the plan get slower?
4. How does the example perform over repeated warm-cache runs, given that execution time went from 2.690 ms to 14.428 ms in the single run shown?
5. How much planning time does the new lookup add for joins on tables that have dependencies statistics, measured over many runs?

## Touch points

- **planner behaviour change:** Row estimates and costs of parameterized inner scans change when dependencies statistics exist, which can change the chosen access path.
- **backpatchability:** The CommitFest entry lists target version "stable", and the patch changes plans for existing statistics objects.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a scan-level path in clausesel.c that later join-level work in eqjoinsel() or calc_joinrel_size_estimate must stay consistent with.
- **performance on the hot path:** Planning now looks up dependencies statistics for cross-relation clause lists; the example shows planning time of 0.110 ms before and 0.648 ms after, over one run.

## Who would commit it, and when

- Owner: qualified committer engaged. Tomas Vondra reviewed in message #2 and supports the general direction. The owner must know both extended statistics and join selectivity estimation.
- Priority: fix design first; audience niche; review cost low. The patch is 4.3 KB in two planner files and helps only users with dependencies statistics on correlated join columns in nested loops. The scope question from message #2 is open, and the scan and join estimates disagree until it is settled.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
