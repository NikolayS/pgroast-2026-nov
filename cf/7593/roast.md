# Design pre-review: Fix initial sync of zero-column published tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7593](https://commitfest.postgresql.org/patch/7593/); [thread](https://www.postgresql.org/message-id/flat/CAL5_ZS1bnYkK_GrA4N4SVKYgYsMMJxL9416m%3DvECAjrcwmQkkg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The thread states the bug with a reproducer, and Peter Smith reproduced it and confirmed the fix in message #3. The fix adds one C entry point used only by tablesync and leaves BeginCopyFrom() unchanged. Confidence is medium because the code was not read and the backpatch range is still open.

Not checked: The code was not read, so the review could not check that the explicit-list flag reaches every place where COPY FROM resolves the column list and prepares defaults. Behavior on back branches is untested.

## Prior attempts

None found before 2026-11-01.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which supported branches show the text-mode bug and which show the binary-mode failure, and does the patch apply and pass its tests on each of them?
2. Should the back branches get the new exported function BeginCopyFromExtended(), or a fix that stays inside tablesync.c, with the API change on master only?
3. Can any other caller of BeginCopyFrom() pass an empty column list and mean "no columns" instead of "all columns"?
4. What should users do with rows that an earlier initial sync already stored with an empty string, and will the release note say so?

## Touch points

- **WAL / replication / standby:** The patch changes tablesync.c, so initial sync of a logical replication subscriber behaves differently for zero-column published tables.
- **ABI / extension API:** The patch adds the exported function BeginCopyFromExtended() to commands/copy.h.
- **backpatchability:** The author asks for a backpatch to all affected branches, but tested master only and the range is not settled.
- **maintenance burden ("who maintains this in 10 years"):** COPY FROM gets a second entry point, BeginCopyFromExtended(), that future changes to BeginCopyFrom() must keep in step.

## Who would commit it, and when

- Owner: silent. A committer who knows logical replication tablesync and the COPY FROM code would have to own it; none has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. The bug needs a zero-column published table and extra subscriber columns, which is rare, but it stores wrong data in text mode and fails in binary mode. The patch is 7.5 KB in four files and reviewed by two people; it waits for a committer and a backpatch decision.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
