# Design pre-review: Use instr_time for pg_stat_database block read/write time counters

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7354](https://commitfest.postgresql.org/patch/7354/); [thread](https://www.postgresql.org/message-id/flat/605732d2-c6bd-4c65-ac83-8d071935d860%40gmx.at), 13 of 13 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible script in message #1, the change follows the instr_time practice that pg_stat_io and pgBufferUsage already use, and it adds no SQL, GUC, catalog or on-disk surface. The open point is sequencing against the planned pgBufferUsage rework, which the thread treats as separate later work.

Not checked: Without the code, nobody here could confirm that PgStat_StatDBEntry and the stats file keep microsecond PgStat_Counter values, or that every flush path converts the accumulated ticks exactly once. Only 3 of 13 messages were visible.

## Prior attempts

None found before 2026-09-28.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Should this patch go in before the planned pgBufferUsage-based rework of pg_stat_database I/O times, or wait for it?
2. Does the patch need a release note or documentation hunk saying that blk_read_time and blk_write_time totals rise after the upgrade?
3. What does the pgbench comparison show with many clients and on another platform, where the clock source differs?
4. Can the vacuum and analyze snapshot of pgStatBlockReadTime and pgStatBlockWriteTime report a wrong elapsed time when track_io_timing changes mid-run?

## Touch points

- **ABI / extension API:** pgstat.h loses pgstat_count_buffer_{read|write}_time, and pgStatBlockReadTime and pgStatBlockWriteTime change type to instr_time.
- **backpatchability:** The header change and the step in reported blk_read_time and blk_write_time values fit a major release only.
- **maintenance burden ("who maintains this in 10 years"):** The planned pgBufferUsage rework would later replace the counters this patch retypes.
- **performance on the hot path:** The patch changes pgstat_count_io_op_time(), which runs on every timed read, write and extend.

## Who would commit it, and when

- Owner: silent. The owner would be a committer who knows the cumulative statistics code and the I/O timing path in pgstat_io.c. The three visible messages show none taking part; the brief reports comments from Andres later in the thread.
- Priority: proceed; audience a sizeable minority; review cost low. The patch touches five files, adds no user-visible surface, and closes a measured gap of about 15% in write time for users who run track_io_timing and compare pg_stat_database with pg_stat_io. The reported speed gain is about 0.5% in one single-client run, so the accuracy fix carries the case.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
