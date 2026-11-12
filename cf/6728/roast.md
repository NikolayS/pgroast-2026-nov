# Design pre-review: md: measure just FileSync() for pgstat_io without FileClose()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6728](https://commitfest.postgresql.org/patch/6728/); [thread](https://www.postgresql.org/message-id/flat/CAKZiRmxwAQ4W-gSJbsox57WsjVOsaZhJE5ZDN2yFo2OOfYckGQ%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2, so nothing ranked stops the patch. The change is confined to md.c and affects only the IOOP_FSYNC timing in pg_stat_io. Confidence is low because the thread has two messages, no measurement of close() time, and the code was not read.

Not checked: Without the code, it could not be checked whether the patch adds a new pgstat_count_io_op_time() variant or moves the FileClose() call. Whether close() time is ever large in this path also could not be judged.

## Prior attempts

None found before 2026-04-30.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. How does the patch take the end time of FileSync() before FileClose() with the current pgstat_count_io_op_time() interface?
2. Is there a measurement where close() time in mdsyncfiletag() changes the IOOP_FSYNC time in pg_stat_io by a visible amount?
3. Can pgaio_closing_fd() block in FileClose() for the file handles that mdsyncfiletag() opens and closes?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch changes how mdsyncfiletag() in md.c times IOOP_FSYNC for pg_stat_io.

## Who would commit it, and when

- Owner: silent. Someone who knows the pg_stat_io timing code and the md.c sync path would have to own it; no committer has spoken in the thread.
- Priority: needs champion; audience niche; review cost low. The patch is about 1 KiB in one file and affects only users who read fsync timing in pg_stat_io with track_io_timing on. It waits on an author reply to message #2 and on a committer who knows the area.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
