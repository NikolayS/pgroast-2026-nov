# Design pre-review: BUG #19772: ERROR: corrupt MVNDistinct entry for GROUP BY c0, ROLLUP (abs(c0)) with ndistinct statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7501](https://commitfest.postgresql.org/patch/7501/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [BUG #19772: ERROR: corrupt MVNDistinct entry for GROUP BY c0, ROLLUP (abs(c0)) with ndistinct statistics](https://www.postgresql.org/message-id/flat/19772-d8c886b4db230132%40postgresql.org), started 2026-10-18
- Messages read: 2 of 2
- Patch prose read: v1-0001-Ignore-grouping-step-nullingrels-when-looking-up-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The patch fixes a plan-time error on valid SQL and extends the nullingrels stripping that e28033fe1 already added to selfuncs.c. The one surviving objection, maintenance-fit-1, is fixable: the back-patch to v18 also changes estimates for queries that run today, and the thread shows no test or comparison for that. A committer who knows the planner has to decide the scope of the back-patch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it is unknown whether the patch strips the GROUP RTE index at every lookup site in selfuncs.c and only there. The size of the estimate change for queries that run today could not be judged.

## The proposal, restated

**Problem.** With an ndistinct extended statistics object, a query like GROUP BY c0, ROLLUP (abs(c0)) fails at plan time with "corrupt MVNDistinct entry". It affects users of extended statistics on v18, v19rc1 and master; the reporter says 14.24 runs the same queries without error. A query fuzzer found it.

**Proposed design.** Since f5050f795, Vars inside a grouping expression missing from some grouping sets carry the GROUP RTE index in their nullingrels. The patch strips that index wherever selfuncs.c already strips outer-join nullingrels before looking up statistics, so estimate_num_groups no longer counts the same column twice. It is back-patched to v18.

**What the author claims.**
- The GROUP RTE bit makes estimate_num_groups add two GroupVarInfos for one column, and estimate_multivariate_ndistinct then reports the error.
- The same bit kept examine_variable from matching such expressions to expression indexes or extended statistics expressions.
- e28033fe1 fixed the same error for outer-join nullingrels.
- The fix belongs in v18 and later, where the GROUP RTE was introduced.

**Thread state.** The bug report and one reply with a v1 patch are visible. The author CCed Richard, who wrote f5050f795 and e28033fe1, but nobody else has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patch changes estimates for queries that did not error

fixable · level 2, rubric item 6 · backpatchability, planner behaviour change · `maintenance-fit-1`

A v18 user runs grouping sets over an expression that has an expression index or extended statistics expression, with no error today. After the minor update, examine_variable matches that statistic, the group estimate changes, and the plan flips. The user finds out from a slower query in production after the upgrade.

The commit message names two effects: the fix for the "corrupt MVNDistinct entry" error, and new matching of expression indexes and extended statistics expressions. Only the first is the reported bug in message #1. The second changes estimates for working queries in a stable branch, and the material shows no test or measurement of it. Stripping the GROUP RTE bit everywhere outer-join bits are stripped does follow existing practice from e28033fe1, so the master change itself is fair.

Thread: [#1 (PG Bug reporting form, 2026-10-18)](https://www.postgresql.org/message-id/19772-d8c886b4db230132%40postgresql.org), [#2 (Matheus Alcantara, 2026-10-20)](https://www.postgresql.org/message-id/CAFY6G8e0CxzGoD7wAukjzeroP7XyBi_sTM_ePLtMFycWRht1LA%40mail.gmail.com)

**Question:** Is the estimate change for queries that do not error intended for v18 back-branches, or can the back-patch be limited to the duplicate-GroupVarInfo fix?

Counter-check: held. The commit message itself names the new expression-index and extended-statistics matching, and the material shows no test or estimate comparison for it. The change probably restores pre-v18 matching, since the GROUP RTE bit did not exist then, but the thread does not say so.

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
| planner behaviour change | **yes** | Stripping the GROUP RTE index in selfuncs.c changes group-count estimates and lets examine_variable match expression indexes and extended statistics expressions. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a back-patch to v18, where estimates for queries that run today may change. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the estimate change for queries that do not error today intended for the v18 back-branch, or can the back-patch be limited to the duplicate-GroupVarInfo fix?
2. Does stripping the GROUP RTE index restore the pre-v18 matching of expression indexes and extended statistics expressions, and can a before-and-after estimate show it?
3. Which of the query variants in message #1 do the new stats_ext tests cover, and does any test cover the expression-index matching?
4. Does any code outside selfuncs.c compare Vars for a statistics lookup in a way that the GROUP RTE index in nullingrels also breaks?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows planner selectivity estimation and the GROUP RTE nullingrels handling; the author CCed the author of f5050f795 and e28033fe1, who has not replied.
- **Liability:** Plan changes in v18 minor releases for grouping-sets queries over expressions, and correct stripping at every statistics lookup site in selfuncs.c, for the life of the v18 branch and later.
- **Author follow-up:** The author answered the report within two days with an analysis and a patch and created the CommitFest entry; no later sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The error needs an ndistinct statistics object and grouping sets over an expression of a grouped column, but it is a regression since v18 on valid SQL. The patch is 7.1 KB, touches selfuncs.c and stats_ext tests, and follows the pattern of e28033fe1.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier attempt at this idea. Message #2 points to e28033fe1 as a fix of the same error for outer-join nullingrels, and this patch extends that approach to the GROUP RTE, so no earlier rejection reason applies.
- standards-compat: 0 objection(s). The patch changes no SQL syntax, GUC, catalog, dump output or psql behaviour, so the user-facing contract is untouched. Back-patching may change group-count and expression-statistics estimates in 18.x, but it moves them back toward pre-v18 behaviour, before the GROUP RTE existed. That makes it a weak case for a plan-stability objection.
- replication-ondisk: 0 objection(s). The fix changes only planner estimation in selfuncs.c and adds regression tests. It touches no on-disk format, WAL record, catalog, wire protocol message, or exported struct, so it has no persistent or cross-node state to object to. Back-patching to 18 carries no pg_upgrade or ABI risk that the commit message shows, although the code was not visible to confirm that no function signature changes.
- security: 0 objection(s). The patch changes only how selfuncs.c compares Vars before it looks up statistics for planner estimates. It adds no privilege, role, setting or user-visible surface, so no new data becomes visible to any user. Better matching to expression indexes and extended statistics still goes through the existing statistics permission and leakproof checks in examine_variable, and the patch does not appear to change those checks. Whether that holds in the code could not be checked here.
- maintenance-fit: 1 objection(s). The master-branch design extends an existing pattern in selfuncs.c and adds no hot-path cost or new API. The author of the related commits is CCed but has not replied yet.
- already-exists: 0 objection(s). This is a bug fix for a planner error. No existing feature, extension or setting avoids it short of dropping the ndistinct statistics object. The patch extends the existing nullingrels stripping in selfuncs.c and adds no second mechanism, so this angle has nothing to raise.
- demand: 0 objection(s). This is a bug fix for a plan-time ERROR on valid SQL, a regression since v18 that 14.24 does not have. A fuzzer finding it does not weaken demand: any user with ndistinct statistics who writes such a grouping-sets query gets an error. Costs stay inside selfuncs.c statistics lookup, with no new user-visible surface.

### Precedents retrieved

- P1: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P2: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P3: [Skip a redundant singleton GROUP BY node](https://commitfest.postgresql.org/patch/7419/) (cf, Withdrawn, 2026-10-09) - judged not relevant; reason summarised from its thread
- P4: [remove_useless_groupby_columns is too enthusiastic](https://commitfest.postgresql.org/patch/3766/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P5: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P6: [Extended statistics / estimate Var op Var clauses](https://commitfest.postgresql.org/patch/2831/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [rewriteGraphTable: Fix missing RTEs in FROM clause by setting inFromCl=true](https://commitfest.postgresql.org/patch/6602/) (cf, Withdrawn, 2026-09-10) - judged not relevant; reason summarised from its thread
- P8: [(SQL/PGQ) cache lookup failed for label](https://commitfest.postgresql.org/patch/6776/) (cf, Withdrawn, 2026-05-15) - judged not relevant; reason summarised from its thread
- P9: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P10: [Revert "Add soft error handling to some expression nodes"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8ec5e0543b90372c8e6d5cc2cd3d2ff89ca0e82) (revert, Reverted, 2023-10-02) - judged not relevant
- P11: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P12: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P13: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P14: [Typos in the comment for the estimate_multivariate_ndistinct()](https://commitfest.postgresql.org/patch/5740/) (cf, Withdrawn, 2025-05-07) - judged not relevant
- P15: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P16: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P17: [Created multiple entries](https://commitfest.postgresql.org/patch/2530/) (cf, Withdrawn, 2020-04-10) - judged not relevant
- P18: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P19: [Allow direct lookups of SpecialJoinInfo by ojrelid](https://commitfest.postgresql.org/patch/4313/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P20: [Reorganize collation lookup time and place](https://commitfest.postgresql.org/patch/1956/) (cf, Returned with feedback, 2019-02-01) - judged not relevant
- P21: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P22: [Issues with 2PC at recovery: CLOG lookups and GlobalTransactionData](https://commitfest.postgresql.org/patch/5532/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P23: [Showing applied extended statistics in explain Part2](https://commitfest.postgresql.org/patch/5077/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Created multiple entries](https://commitfest.postgresql.org/patch/2531/) (cf, Withdrawn, 2020-04-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 92 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 46 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:01:58.093Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
