# Design pre-review: Discard aborted updaters when expanding a multixact

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7304](https://commitfest.postgresql.org/patch/7304/); [thread](https://www.postgresql.org/message-id/flat/5778D6297D3CCF21%2B202609151836512678486%40vastdata.com.cn), 9 of 9 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The thread states the problem with a concrete interleaving, an independent pgbench reproducer on released versions (message #7) and a 2022 report with the same error. The fix changes existing conflict checks in heapam.c and adds no new user-visible surface. Confidence is medium because the code and the final commit were not read.

Not checked: The code and the final commit were not read, so the claim that every caller of the changed conflict check tolerates the added wait is unverified. The remaining TransactionIdDidAbort() use in heap_lock_updated_tuple_rec() rests on the author's analysis in message #3.

## Prior attempts

None found before 2026-09-15.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches received the injection point in xact.c and the new isolation and recovery tests?
2. Is there a measurement of the extra wait for conflicting requests between the pg_xact abort write and ProcArray cleanup under high concurrency?
3. Should heap_lock_updated_tuple_rec() also test that TransactionIdDidCommit() returns false, so that it does not lock a dead successor of a crashed updater?
4. Does the CommitFest entry need its status changed from Needs review to Committed?

## Touch points

- **backpatchability:** The thread reports a back-patch to all supported versions, and the tests add an injection point in xact.c.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes multixact updater checks in heapam.c, an area few people know well.
- **performance on the hot path:** Conflicting requests now wait until the aborting updater leaves ProcArray; the thread shows no measurement of that wait.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas is reviewer and committer on the CommitFest entry, rewrote the comments as v5, and reports the commit and back-patch in the thread.
- Priority: proceed; audience a sizeable minority; review cost medium. The bug hits workloads that mix foreign-key locks and rolled-back updates, with 43 of 48 pgbench clients failing in the report on REL_18_STABLE. The code patches are small and add no new surface, but the heapam.c multixact logic needs a reviewer who knows it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
