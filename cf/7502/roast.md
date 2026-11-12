# Design pre-review: Fix rescan costing of single-batch hash joins

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7502](https://commitfest.postgresql.org/patch/7502/); [thread](https://www.postgresql.org/message-id/flat/CA%2Bfm-RMQd9rvbwQD4aL4iAYSo3869DPH8_CX%2BBGBCS91aKQ-og%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. The problem is stated with two reproducible scripts, and Etsuro Fujita agreed in message #2 that the costing does not match ExecReScanHashJoin(). The confidence is low because the panel raised no objections and the patch code was not read, so the result rests on the prose alone.

Not checked: Without the code, nobody checked how the outer rescan charge and the parameterized-inner charge combine, or whether parallel and multi-batch hash joins are affected. The thread reports no plan-change results beyond the regression suite and the two scripts.

## Prior attempts

None found before 2026-10-21.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. When the inner path is parameterized, does the full join cost replace the outer path's rescan cost, or can both charges apply to one join?
2. Which plans change on a join-heavy benchmark such as TPC-H or the join order benchmark, and do any of them run slower with the patch?
3. Does the new charge apply to parallel hash joins and to multi-batch hash joins, or do those cases keep their current rescan cost?
4. How many prepared statements or SQL functions that are fast on custom plans today would move to a slower generic plan with the higher custom plan cost?
5. Does merge join or another node handled by cost_rescan() have the same underestimate when its input reruns on every rescan?

## Touch points

- **planner behaviour change:** cost_rescan() in costsize.c charges more for rescans of single-batch hash joins, so nested loop plans over hash joins can change.
- **backpatchability:** The entry is tagged Bugfix, but author and reviewer agree it is master only because it changes plans.

## Who would commit it, and when

- Owner: qualified committer engaged. Etsuro Fujita agreed with the diagnosis in message #2 and signed up as reviewer; the owner must know planner costing and the hash join rescan path in the executor.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is 6.2 KB, touches costsize.c and one regression test, and adds no user-visible surface. It affects LATERAL queries and SQL functions on the plan cache in 18, and a planner-area committer plans to review it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
