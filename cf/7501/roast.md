# Design pre-review: BUG #19772: ERROR: corrupt MVNDistinct entry for GROUP BY c0, ROLLUP (abs(c0)) with ndistinct statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7501](https://commitfest.postgresql.org/patch/7501/); [thread](https://www.postgresql.org/message-id/flat/19772-d8c886b4db230132%40postgresql.org), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The patch fixes a plan-time error on valid SQL and extends the nullingrels stripping that e28033fe1 already added to selfuncs.c. The one surviving objection, maintenance-fit-1, is fixable: the back-patch to v18 also changes estimates for queries that run today, and the thread shows no test or comparison for that. A committer who knows the planner has to decide the scope of the back-patch.

Not checked: The code was not read, so it is unknown whether the patch strips the GROUP RTE index at every lookup site in selfuncs.c and only there. The size of the estimate change for queries that run today could not be judged.

## Prior attempts

None found before 2026-10-18.

## Objections a reviewer is likely to raise

### 1. Back-patch changes estimates for queries that did not error

A v18 user runs grouping sets over an expression that has an expression index or extended statistics expression, with no error today. After the minor update, examine_variable matches that statistic, the group estimate changes, and the plan flips. The user finds out from a slower query in production after the upgrade.

The commit message names two effects: the fix for the "corrupt MVNDistinct entry" error, and new matching of expression indexes and extended statistics expressions. Only the first is the reported bug in message #1. The second changes estimates for working queries in a stable branch, and the material shows no test or measurement of it. Stripping the GROUP RTE bit everywhere outer-join bits are stripped does follow existing practice from e28033fe1, so the master change itself is fair.

**Question:** Is the estimate change for queries that do not error intended for v18 back-branches, or can the back-patch be limited to the duplicate-GroupVarInfo fix?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/19772-d8c886b4db230132%40postgresql.org), [#2](https://www.postgresql.org/message-id/CAFY6G8e0CxzGoD7wAukjzeroP7XyBi_sTM_ePLtMFycWRht1LA%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Is the estimate change for queries that do not error today intended for the v18 back-branch, or can the back-patch be limited to the duplicate-GroupVarInfo fix?
2. Does stripping the GROUP RTE index restore the pre-v18 matching of expression indexes and extended statistics expressions, and can a before-and-after estimate show it?
3. Which of the query variants in message #1 do the new stats_ext tests cover, and does any test cover the expression-index matching?
4. Does any code outside selfuncs.c compare Vars for a statistics lookup in a way that the GROUP RTE index in nullingrels also breaks?

## Touch points

- **planner behaviour change:** Stripping the GROUP RTE index in selfuncs.c changes group-count estimates and lets examine_variable match expression indexes and extended statistics expressions.
- **backpatchability:** The commit message asks for a back-patch to v18, where estimates for queries that run today may change.

## Who would commit it, and when

- Owner: silent. Someone who knows planner selectivity estimation and the GROUP RTE nullingrels handling; the author CCed the author of f5050f795 and e28033fe1, who has not replied.
- Priority: proceed; audience niche; review cost low. The error needs an ndistinct statistics object and grouping sets over an expression of a grouped column, but it is a regression since v18 on valid SQL. The patch is 7.1 KB, touches selfuncs.c and stats_ext tests, and follows the pattern of e28033fe1.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
