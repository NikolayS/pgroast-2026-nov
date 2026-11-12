# Design pre-review: Allow subquery pull-up past inlineable CTEs

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7283](https://commitfest.postgresql.org/patch/7283/); [thread](https://www.postgresql.org/message-id/flat/15507998da7eff34854a5de3494fe9d8%40postgrespro.ru), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Only demand-1 survived, and it is fixable. The thread shows no real-world query, but the extra planning work falls only on subqueries that were rejected before, and the existing code marks this gap with an XXX comment. So no level fails. What remains is a planner committer's judgment on plan changes and on the cases the patch prose leaves open, such as recursive CTEs.

Not checked: Without the code, the handling of recursive CTEs, CTEs referenced from nested sublevels, and permission info of inlined CTEs could not be checked. Only 3 of 6 thread messages were visible.

## Prior attempts

None found before 2026-09-07.

## Objections a reviewer is likely to raise

### 1. No real-world query shown that needs a WITH inside a pulled-up subquery

A committer takes on new prepjointree.c logic, kept in step with SS_process_ctes() through is_cte_inlineable(), for a case shown only on regression tables. If few users write WITH inside a subquery, the project pays for that coupling and gets little back.

Every example in messages #1 and #3 is synthetic, and message #2 found that the first one did not reproduce. No field report, -general or -bugs thread, or production workload appears in the material. Hand-written queries can move the WITH to the top level, where it is already inlined. [P8] was declined on the same ground, though there the cost fell on everyone and here it mostly does not.

Precedent: [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (Rejected, 2018-11-23)

**Question:** Which real queries, for example views defined with WITH or ORM-generated SQL used under outer joins, hit this today, and why can their authors not move the WITH to the top level?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/15507998da7eff34854a5de3494fe9d8%40postgrespro.ru), [#2](https://www.postgresql.org/message-id/CAL9_%2BFFThoygHo91V32t5N7oLF1otuzUCg5m%3Dmcur9BsaK713A%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/2f11cdbdd5d905c0a01ef2b2e26e3034%40postgrespro.ru); `demand-1`</sub>

## Questions for the author

1. Which real queries, such as views defined with WITH or ORM-generated SQL under outer joins, hit this restriction today?
2. How does the pull-up path handle recursive CTEs and CTEs referenced from a nested sublevel of the subquery?
3. What is the planning-time cost for a subquery with many or large CTEs, including the case where pull-up is abandoned at the recheck?
4. Have you found any existing query whose plan gets worse after the subquery is pulled up?
5. Do the with.sql tests now use the tenk1/tenk2 example from message #3 in place of the one that did not reproduce on master?

## Touch points

- **ABI / extension API:** subselect.h gains new exported helpers (SS_all_ctes_inlineable, SS_inline_ctes); no existing signature changes.
- **planner behaviour change:** Subqueries with inlineable CTEs are now pulled up, so plans of existing queries can change.
- **maintenance burden ("who maintains this in 10 years"):** CTE inlining now also runs inside pull_up_simple_subquery, a second call site next to SS_process_ctes.
- **performance on the hot path:** is_simple_subquery now walks each CTE query tree for subqueries that have a WITH; no planning-time numbers are shown.

## Who would commit it, and when

- Owner: silent. Someone who knows planner preprocessing: subquery pull-up in prepjointree.c and CTE handling in subselect.c. No committer has spoken in the visible thread.
- Priority: proceed; audience unclear; review cost medium. The patch is about 14 KiB, adds no user-visible surface and removes a restriction the code already marks with an XXX comment. The thread shows only regression-table examples, so the audience is not known, and planner pull-up code needs a careful read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
