# Design pre-review: Allow subquery pull-up past inlineable CTEs

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7283](https://commitfest.postgresql.org/patch/7283/) (Needs review); authors: Andrey Kazarinov (kazar)
- Thread: [\[PATCH\] Allow subquery pull-up past inlineable CTEs](https://www.postgresql.org/message-id/flat/15507998da7eff34854a5de3494fe9d8%40postgrespro.ru), started 2026-09-07
- Messages read: 6 of 6
- Patch prose read: v2-0001-Allow-subquery-pull-up-past-inlineable-CTEs.patch, 0001-Allow-subquery-pull-up-past-inlineable-CTEs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Only demand-1 survived, and it is fixable. The thread shows no real-world query, but the extra planning work falls only on subqueries that were rejected before, and the existing code marks this gap with an XXX comment. So no level fails. What remains is a planner committer's judgment on plan changes and on the cases the patch prose leaves open, such as recursive CTEs.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the handling of recursive CTEs, CTEs referenced from nested sublevels, and permission info of inlined CTEs could not be checked. Only 3 of 6 thread messages were visible.

## The proposal, restated

**Problem.** is_simple_subquery() rejects any subquery with a non-empty cteList, even after SS_process_ctes has inlined every CTE. The subquery is then planned separately, so the outer query cannot use join conditions inside it, for example a parameterized index scan through an outer join. This affects queries that put an inlineable WITH inside a subquery.

**Proposed design.** The patch moves the CTE inlining checks from SS_process_ctes() into is_cte_inlineable(). is_simple_subquery() accepts a subquery whose CTEs all pass that check, and rejects one with any unreferenced SELECT CTE. pull_up_simple_subquery() inlines the CTEs on its copy of the subquery and clears cteList, so the original RTE stays intact if pull-up is abandoned.

**What the author claims.**
- A tenk1/tenk2 example changes from a hash join with a full seq scan to a nested loop with 10 index lookups.
- Extra planning cost hits only subqueries that were rejected before, so the pull-up gain pays for it.
- MATERIALIZED, volatile, multiply-referenced and unreferenced CTEs still block pull-up.
- An existing with.sql plan loses a Subquery Scan, which the author calls the expected improvement.

**Thread state.** Only reviewer Xinlei (Clemenza) Zhang replied; she tested v2, found results identical to the equivalent no-CTE query, and reported no new issues. No committer has commented, and the CommitFest entry is in Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-07; 23 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No real-world query shown that needs a WITH inside a pulled-up subquery

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), planner behaviour change · `demand-1`

A committer takes on new prepjointree.c logic, kept in step with SS_process_ctes() through is_cte_inlineable(), for a case shown only on regression tables. If few users write WITH inside a subquery, the project pays for that coupling and gets little back.

Every example in messages #1 and #3 is synthetic, and message #2 found that the first one did not reproduce. No field report, -general or -bugs thread, or production workload appears in the material. Hand-written queries can move the WITH to the top level, where it is already inlined. [P8] was declined on the same ground, though there the cost fell on everyone and here it mostly does not.

**Precedents.**
- [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (Rejected, 2018-11-23); thread: [link](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)

Thread: [#1 (Andrey Kazarinov, 2026-09-07)](https://www.postgresql.org/message-id/15507998da7eff34854a5de3494fe9d8%40postgrespro.ru), [#2 (Clemenza Zhang, 2026-09-07)](https://www.postgresql.org/message-id/CAL9_%2BFFThoygHo91V32t5N7oLF1otuzUCg5m%3Dmcur9BsaK713A%40mail.gmail.com), [#3 (Andrey Kazarinov, 2026-09-08)](https://www.postgresql.org/message-id/2f11cdbdd5d905c0a01ef2b2e26e3034%40postgrespro.ru)

**Question:** Which real queries, for example views defined with WITH or ORM-generated SQL used under outer joins, hit this today, and why can their authors not move the WITH to the top level?

Counter-check: held. The thread shows only regression-table examples and no user report, and a view defined with WITH is a plausible case but is not shown. [P8] must not be used as support, because there the cost fell on every query.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | subselect.h gains new exported helpers (SS_all_ctes_inlineable, SS_inline_ctes); no existing signature changes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Subqueries with inlineable CTEs are now pulled up, so plans of existing queries can change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | CTE inlining now also runs inside pull_up_simple_subquery, a second call site next to SS_process_ctes. |
| performance on the hot path | **yes** | is_simple_subquery now walks each CTE query tree for subqueries that have a WITH; no planning-time numbers are shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which real queries, such as views defined with WITH or ORM-generated SQL under outer joins, hit this restriction today?
2. How does the pull-up path handle recursive CTEs and CTEs referenced from a nested sublevel of the subquery?
3. What is the planning-time cost for a subquery with many or large CTEs, including the case where pull-up is abandoned at the recheck?
4. Have you found any existing query whose plan gets worse after the subquery is pulled up?
5. Do the with.sql tests now use the tenk1/tenk2 example from message #3 in place of the one that did not reproduce on master?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows planner preprocessing: subquery pull-up in prepjointree.c and CTE handling in subselect.c. No committer has spoken in the visible thread.
- **Liability:** Plan changes and any wrong-result bug for queries with a WITH inside a pulled-up subquery, plus keeping the two inlining call sites in step, for as long as the code exists.
- **Author follow-up:** The author posted v2 one day after the review and said he will continue from another email address; the entry is tagged My First Patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch is about 14 KiB, adds no user-visible surface and removes a restriction the code already marks with an XXX comment. The thread shows only regression-table examples, so the audience is not known, and planner pull-up code needs a careful read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Pull-up gate relies on SS_process_ctes behavior that it does not share. Refuted (already handled): The patch already puts the inlining decision in is_cte_inlineable(), which both paths call, so a new materialization option reaches both. The separate cterefcount == 0 rule only blocks pull-up, so a later change there cannot drop a CTE.

### Review angles

- prior-attempts: 0 objection(s). No precedent matches this idea. [P8]'s demand concern applies only weakly, because views defined with WITH produce this shape and only such subqueries pay the planning cost. [P5] died over duplicated code, while this patch moves the inlining checks into one helper. The thread cites no earlier discussion beyond the existing XXX comment quoted in message #1.
- standards-compat: 0 objection(s). The patch changes no syntax, GUC, catalog, dump format or psql output. Plans can change for existing queries that put a default or NOT MATERIALIZED CTE inside a subquery, but that matches what the docs already promise for inlineable CTEs, and MATERIALIZED and OFFSET 0 still work as optimization fences. Data-modifying CTEs are already rejected below the top level, and the patch prose does not say how recursive CTEs are handled, which only the code can show.
- replication-ondisk: 0 objection(s). The proposal changes only planner behaviour in subselect.c and prepjointree.c. It adds no on-disk format, WAL record, catalog column, catversion bump or protocol message, and stored view definitions keep their form. The new subselect.h helpers add symbols and change no existing struct or signature, so extensions are not affected.
- security: 0 objection(s). The patch only changes planner join-tree flattening for queries the user already has permission to run, and it adds no privilege, role, GUC or catalog surface. Subqueries marked security_barrier and RLS security quals keep their existing pull-up limits, and permission checks move up with the pulled-up range table as they do for other pull-ups. Whether the inlined CTEs' permission info merges correctly can only be checked in the code.
- maintenance-fit: 1 objection(s). From this angle the patch is small and general. It removes a long-standing XXX restriction, and the extra planning work runs only for subqueries that were rejected before. No committer has engaged yet, and the author is new to the project, so ownership after commit is open. That is a committability issue, not an objection to the idea.
- already-exists: 0 objection(s). Postgres has no existing feature, extension or hook that removes this subquery boundary. The only workaround is a manual query rewrite, either moving the WITH to the top level or dropping it, and that is not a feature. The patch reuses the existing inlining checks through a shared is_cte_inlineable() and does not add a second mechanism. The existing code also marks the gap with an XXX comment.
- demand: 1 objection(s). Only a show-demand objection applies. The extra planning cost falls only on subqueries that were rejected before, so who-pays-versus-who-gains does not fail here.

### Precedents retrieved

- P1: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant; reason summarised from its thread
- P2: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P3: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P4: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P5: [Pulling up direct-correlated ANY_SUBLINK](https://commitfest.postgresql.org/patch/3941/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P6: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P7: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P8: [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (cf, Rejected, 2018-11-23) - used in the review; reason summarised from its thread
- P9: [Make transformAExprIn() return a flattened bool expression directly](https://commitfest.postgresql.org/patch/6723/) (cf, Withdrawn, 2026-07-11) - judged not relevant; reason summarised from its thread
- P10: [Use simplehash.h instead of dynahash in SMgr](https://commitfest.postgresql.org/patch/3092/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P11: [Simplifying the interface of UpdateMinRecoveryPoint](https://commitfest.postgresql.org/patch/673/) (cf, Rejected, 2016-08-26) - judged not relevant
- P12: [Optimization for lazy_scan_heap](https://commitfest.postgresql.org/patch/699/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P13: [Document atthasmissing default optimization avoids verification table scan](https://commitfest.postgresql.org/patch/3329/) (cf, Withdrawn, 2022-03-29) - judged not relevant
- P14: [optimizing vacuum truncation scans WIP](https://commitfest.postgresql.org/patch/220/) (cf, Returned with feedback, 2015-08-12) - judged not relevant
- P15: [Simplify documentation related to Windows builds](https://commitfest.postgresql.org/patch/4745/) (cf, Withdrawn, 2024-05-16) - judged not relevant
- P16: [Change simple_heap_insert() to a macro](https://commitfest.postgresql.org/patch/1826/) (cf, Rejected, 2018-11-02) - judged not relevant
- P17: [\[PATCH\] Add support function for containment operators](https://commitfest.postgresql.org/patch/4304/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P18: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P19: [remove deprecated v8.2 containment operators](https://commitfest.postgresql.org/patch/2798/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P20: [Corresponding by clause for set operation](https://commitfest.postgresql.org/patch/1016/) (cf, Returned with feedback, 2017-04-04) - judged not relevant
- P21: [Contains and is contained by operators of inet datatypes](https://commitfest.postgresql.org/patch/805/) (cf, Rejected, 2016-11-30) - judged not relevant
- P22: [OR-clauses for index](https://commitfest.postgresql.org/patch/454/) (cf, Returned with feedback, 2016-03-29) - judged not relevant
- P23: [Removing useless DISTINCT clauses](https://commitfest.postgresql.org/patch/1369/) (cf, Returned with feedback, 2018-04-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 115 s; cost reported by the CLI: $1.26
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 23 kept
- Tone check: 52 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:37:17.317Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
