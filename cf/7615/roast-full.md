# Design pre-review: Eliminate redundant outer-join clone clauses after constant propagation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7615](https://commitfest.postgresql.org/patch/7615/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [Eliminate redundant outer-join clone clauses after constant propagation](https://www.postgresql.org/message-id/flat/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com), started 2026-11-09
- Messages read: 1 of 1
- Patch prose read: 0001-Remove-redundant-constant-propagated-outer-join-clau.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The proposal does not name a target branch, and the answer decides the design (standards-compat-1). ABI stability is the only reason given to avoid the v3 RestrictInfo field, and that limit applies only to back branches. Message #1 also shows no query or timing that got worse after f4002899 (demand-1). Both are fixable, but a committer would likely ask for them before reading the nullability check.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it could not be judged whether the nullability check covers every deferred pushed-down outer join case or what it costs at planning time. The two level 2 objections on norm_selec reuse and the duplicated nullability reasoning were not evaluated.

## The proposal, restated

**Problem.** Commit f4002899 fixed a wrong-result bug by keeping every constant-propagated outer-join clone clause, marked with norm_selec > 1. This leaves unnecessary join filters in join orders where the clause is redundant. An earlier fix (v3) avoided this by adding a RestrictInfo field, but that breaks ABI and cannot be back-patched.

**Proposed design.** When the planner builds the restriction list for a join or parameterized path, it omits clone clauses marked norm_selec > 1 unless a deferred, pushed-down outer join may already have null-extended the clause's outer operand. The check uses existing RestrictInfo and SpecialJoinInfo fields only. The clause stays distributed and available during join-order search.

**User-visible surface.**
- Plan changes: some queries with outer joins and constant-propagated equalities lose redundant join filters and may get different join orders.

**What the author claims.**
- Restores removal of unnecessary join filters that f4002899 left in place.
- Keeps the correctness fix from f4002899.
- Adds no field to RestrictInfo, so the ABI does not change.
- Regression test changes are plan-only.

**Thread state.** Only the first message is visible. No one has replied yet.

## 2. Prior attempts

### Redundant qualifier elimination (related)

- **Link:** [https://commitfest.postgresql.org/patch/7127/](https://commitfest.postgresql.org/patch/7127/)
- **Thread:** [https://www.postgresql.org/message-id/flat/DKBX1KUW8KUW.2DU5RD192DHIQ@gmail.com](https://www.postgresql.org/message-id/flat/DKBX1KUW8KUW.2DU5RD192DHIQ@gmail.com)
- **Date:** decided 2026-08-14; started 2026-07-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Denis Smirnov (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane opposed the patch even with fixes, because it adds roughly O(N^2) planning cost in the number of restriction clauses to every query to handle badly written queries. Denis Smirnov showed that removing a duplicate volatile clause such as nextval('s') = 1 changes query results, and that the patch does not fix the JIT regression in the original report. The author accepted the feedback.
- **Reason (quote):** "It would impose nontrivial costs on every query --- roughly O(N^2) in the number of restriction clauses --- in order to fix badly-written queries, and that is not a tradeoff I like." ([message](https://www.postgresql.org/message-id/3766950.1786724867%40sss.pgh.pa.us))
- **What is different this time:** P4 compared every base restriction clause against all others with equal() for every query. This patch tests only clone clauses already marked norm_selec > 1 when it builds a join or parameterized-path restriction list.


## 3. Objections a reviewer is likely to raise

### 1. Plan-changing optimization framed around back-branch ABI limits

fixable · level 1b, rubric item 6 · backpatchability, planner behaviour change · `standards-compat-1`

A user on a stable branch applies a minor release that contains this patch. A query with outer joins and a constant-propagated equality gets a different join order and runs slower. The user finds out from production latency after an upgrade that was expected to be safe.

Message #1 gives ABI stability as the only reason to avoid the v3 RestrictInfo field, and that constraint applies only to back-branches. The proposal does not say whether it targets master or stable branches. Postgres does not usually back-patch planner optimizations that change plans, and join.out shows the plans change. On master, a new field is allowed, so the ABI argument for this design does not hold there.

Thread: [#1 (Tender Wang, 2026-11-09)](https://www.postgresql.org/message-id/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com)

**Question:** Is this patch meant for back-branches, and if it is only for master, why not use the clearer v3 RestrictInfo field instead?

Counter-check: held. Message #1 gives ABI as the only reason to avoid the v3 field and does not name a target branch. The ABI limit applies only to back branches, so the question stands.

### 2. No evidence the leftover join filters cost any user anything

fixable · level 1b, rubric item 3 · demand, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `demand-1`

A committer adds a second, subtle nullability check to clause selection in relnode.c, in the area where f4002899 just fixed a wrong-result bug. The only gain is dropping a redundant equality filter, because the derived INNERVAR = CONSTANT already applies. If the new check misses a commuting-join case, users get wrong results, and they find out from bug reports.

Message #1 gives no field report, no -general or -bugs thread and no query that got slower after f4002899. It also gives no execution or planning-time numbers. The only evidence is changed plans in join.out, which show the plan differs but not that it is faster. With f4002899, the leftover clause adds no selectivity, so the main remaining cost is one cheap filter per row.

Thread: [#1 (Tender Wang, 2026-11-09)](https://www.postgresql.org/message-id/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com)

**Question:** Which real queries got measurably slower after f4002899, and by how much does this patch improve their planning or execution time?

Counter-check: held. The patch restores an optimization that existed before f4002899, and the join.out changes include different join orders, so the gain may be more than one filter. But message #1 shows no query or timing, so the request for numbers stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The design avoids a new RestrictInfo field for ABI reasons and gives norm_selec > 1 a second meaning. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | join.out shows dropped join filters and different join orders for outer-join queries. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 presents ABI safety as the gain over v3 but does not name a target branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | relnode.c gains a separate outer-join nullability check built from six RestrictInfo and SpecialJoinInfo fields. |
| performance on the hot path | **yes** | The check runs when the planner builds join and parameterized-path restriction lists; no planning-time numbers are given. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is this patch meant for back branches or for master only?
2. If it targets master only, why is reusing norm_selec > 1 better than the explicit RestrictInfo field from v3?
3. Which queries got measurably slower after f4002899, and how much does this patch improve their planning or execution time?
4. Which regression test exercises the deferred pushed-down outer join case, so that a later commutation change that breaks the check fails visibly?
5. What stops other planner code from setting norm_selec above 1 on a clause that must still be evaluated?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows outer-join commutation, clone clauses and incompatible_relids handling in the planner. No one has replied to message #1.
- **Liability:** Wrong results for outer-join queries if the check misses a deferral case, on every branch that receives the patch. Later commutation changes must keep this check in step.
- **Author follow-up:** The author wrote v3 for the original wrong-result bug and returned with this follow-up, which suggests continued involvement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is small and touches one planner file, but it sits where f4002899 just fixed a wrong-result bug. The target branch must be settled first, because it decides whether reusing norm_selec is justified.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-1` (fixable, level 2) Correctness depends on norm_selec > 1 used as a semantic flag. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Second copy of outer-join nullability reasoning in relnode.c. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-3` Back-patch shaped design for a plan-changing optimization. Refuted (not specific): Duplicate of standards-compat-1. Its request for numbers is covered by demand-1.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent is the same idea. Message #1 reports one earlier attempt, v3 for the original wrong-result bug, which was set aside because a new RestrictInfo field breaks ABI. This patch adds no field, so that reason no longer applies. The P4 concern about planning cost could matter if the new check runs for every join restriction list, but the material gives no sign of quadratic cost, so I raise no objection from prior attempts.
- standards-compat: 1 objection(s). Apart from the release-target question, the user-facing contract does not change: no syntax, no GUC, no catalog or pg_dump impact, and results stay the same by design.
- replication-ondisk: 0 objection(s). The patch changes only planner logic in relnode.c and plan output in join.out. It adds no catalog, WAL, on-disk, wire-protocol or struct change, and it keeps RestrictInfo unchanged, so nothing persists or crosses a process boundary. Back-branch plan stability, if back-patching is intended, belongs to another angle.
- security: 0 objection(s). The patch only changes which clone clauses the planner keeps in a join or parameterized path restriction list. It adds no privilege, role, view, log output or user-controlled input path, so this angle has no objection. A wrong plan here would give wrong results, which is a correctness concern and outside the security angle.
- maintenance-fit: 3 objection(s). No committer has replied yet, so who would own this optimizer code is open. [P4] covers a different duplicate-qual check, and its O(N^2) cost reason does not apply here.
- already-exists: 0 objection(s). No existing feature, setting or extension removes redundant clone clauses in planner internals, so this angle has nothing to raise. The patch refines the norm_selec > 1 marker from f4002899 instead of adding a second mechanism. [P4] covers a different case: duplicate base quals and their O(N^2) cost.
- demand: 1 objection(s). Demand is the only open point from this angle. The plan changes alone do not show that users gain anything.

### Precedents retrieved

- P1: [Disallow outer-level and WHERE-clause aggregates in GRAPH_TABLE](https://commitfest.postgresql.org/patch/7144/) (cf, Withdrawn, 2026-09-08) - judged not relevant; reason summarised from its thread
- P2: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P3: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant; reason summarised from its thread
- P4: [Redundant qualifier elimination](https://commitfest.postgresql.org/patch/7127/) (cf, Returned with feedback, 2026-08-14) - used in the review; reason summarised from its thread
- P5: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P6: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant; reason summarised from its thread
- P7: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P8: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P9: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P10: [Plan a filtering inner join as a semijoin](https://commitfest.postgresql.org/patch/7202/) (cf, Withdrawn, 2026-08-27) - judged not relevant
- P11: [allow specifying action when standby encounters incompatible parameter settings](https://commitfest.postgresql.org/patch/3619/) (cf, Withdrawn, 2022-07-25) - judged not relevant
- P12: [Push down aggregates below joins.](https://commitfest.postgresql.org/patch/1685/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant
- P14: [Rename constants in preparation of variableness](https://commitfest.postgresql.org/patch/4828/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P15: [Remove redundant DISTINCT when GROUP BY already guarantees uniqueness](https://commitfest.postgresql.org/patch/6869/) (cf, Withdrawn, 2026-07-23) - judged not relevant
- P16: [Bug: wrong relname in RemoveSubscriptionRel() error detail](https://commitfest.postgresql.org/patch/6621/) (cf, Withdrawn, 2026-07-12) - judged not relevant
- P17: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant
- P18: [add DETAIL for wrong SCRAM password](https://commitfest.postgresql.org/patch/3022/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/1247/) (cf, Withdrawn, 2021-03-11) - judged not relevant
- P20: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/3764/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P21: [Table Partition + Join Pushdown](https://commitfest.postgresql.org/patch/325/) (cf, Rejected, 2016-01-25) - judged not relevant
- P22: [hash joins with bloom filters](https://commitfest.postgresql.org/patch/1540/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P23: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant
- P24: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 152 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 72 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:07:57.224Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
