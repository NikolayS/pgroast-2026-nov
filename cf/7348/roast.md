# Design pre-review: Table sync race with REFRESH PUBLICATION

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7348](https://commitfest.postgresql.org/patch/7348/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWWJfq%3DM5U9h7sJFcQ%2BLPwq76CzLTzQbkRQPTJJ47SGxWw%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproduction, and the fix re-reads one pg_subscription_rel state under a lock the apply worker already takes, following the sequence-side fix in dca73f7dd03. Confidence is medium because no code was read and only one reviewer has confirmed that the state check is enough without the LSN check.

Not checked: No code was read. Two points depend on it: whether the state check alone covers the remove and re-add case, and whether the refresh drops the origin and slot that the apply worker now skips.

## Prior attempts

- [Fix alter subscription concurrency errors](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com) (related; Withdrawn, 2023-03-28). "I think the patch as-submitted is pretty uninteresting, mainly because the design of adding bespoke lock code for subscription objects doesn't scale." ([message](https://www.postgresql.org/message-id/956468.1679335244%40sss.pgh.pa.us)) Different now: P3 added new lock-and-retry lookup code for subscription DDL, modeled on dbcommands.c. This proposal adds no lock code: it re-reads one pg_subscription_rel state under the existing lock in the apply worker, following the sequence-side fix in dca73f7dd03.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches have the race, and which branches does the fix target for back-patching?
2. When the apply worker skips origin cleanup, does ALTER SUBSCRIPTION ... REFRESH PUBLICATION still drop the tablesync origin and slot for the removed table?
3. Can a re-added table reach SYNCDONE again while the apply worker still waits for the subscription lock, which would make the state check alone insufficient?
4. Should the remove and re-add case have an automated test, given that it protects against a skipped initial copy?
5. If the test patch is not committed, what keeps this recheck from being removed by a later change to ProcessSyncingTablesForApply()?

## Touch points

- **WAL / replication / standby:** The fix changes how the logical replication apply worker moves a table from SYNCDONE to READY in tablesync.c.
- **catalog changes:** The apply worker re-reads the pg_subscription_rel state under the subscription lock; the catalog schema does not change.
- **backpatchability:** The CommitFest entry targets stable branches, but the thread does not name the affected branches.
- **maintenance burden ("who maintains this in 10 years"):** v3-0002 adds an injection point in tablesync.c that would ship if the test patch were committed.

## Who would commit it, and when

- Owner: silent. Someone who knows the logical replication tablesync and apply worker code. The visible thread shows one reviewer, Peter Smith, who asked for more reviewers.
- Priority: proceed; audience niche; review cost low. The race needs a REFRESH PUBLICATION at a narrow moment and has no field report, but with disable_on_error it disables the whole subscription. The fix is about 2.4 KB in one file and follows an existing pattern, so a second review by someone who knows tablesync.c is cheap.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
