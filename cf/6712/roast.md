# Design pre-review: [Patch]Incorrect usage of PageGetFreeSpace instead of PageGetHeapFreeSpace in heap_xlog_visible

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6712](https://commitfest.postgresql.org/patch/6712/); [thread](https://www.postgresql.org/message-id/flat/66c011a7-2f26-4ab9-a5b0-1afcb8523367%40postgrespro.ru), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patch replaces one function call in heap_xlog_visible so that standby redo records the same FSM value the primary does, and PG 19 already has the fix per message #1. The remaining points are back-patch details: the branch range, the PG 18 variant, and the rebase CFbot asks for. Confidence is medium because only one message is visible and no code was read.

Not checked: The code was not read, so the claim that heap_xlog_visible is the only heap caller of PageGetFreeSpace is unchecked. The PG 18 variant and the cost of the extra line-pointer check during redo could not be judged.

## Prior attempts

None found before 2026-04-23.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Should the fix go to all branches from 14 to 18, and can a PG 18 patch be attached to this thread?
2. Against which branch does the attached patch apply, given that CFbot reports it needs a rebase?
3. What does an inserter on a promoted standby do with the wrong FSM entry, and what does that cost in practice?
4. Can the shell test case become a TAP test, or is the fix meant to go in without a test?

## Touch points

- **WAL / replication / standby:** The patch changes the free space value that heap_xlog_visible writes to the standby FSM during redo.
- **backpatchability:** The fix targets back branches only, PG 14 to 17, with a separate variant needed for PG 18.

## Who would commit it, and when

- Owner: silent. A committer who knows heap redo and the free space map would have to own it; nobody has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. Message #1 calls the mismatch rare, and it affects only standbys with heap pages at the MaxHeapTuplesPerPage limit. The patch is a one-call change in heapam.c, but nobody has replied and the entry moved CommitFests without review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
