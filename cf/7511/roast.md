# Design pre-review: A bug in recovery prefetcher, triggering asserts / FATAL: checkpoint request failed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7511](https://commitfest.postgresql.org/patch/7511/); [thread](https://www.postgresql.org/message-id/flat/a797f236-286a-43e6-9b82-5f3d70bd5e99%40vondra.me), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The thread states the bug, its two failure modes and the fix, and three people confirmed the fix in messages #2 to #4. The patch clears one field of the prefetcher's private state and adds no user-visible surface. Confidence is medium because the code was not read and the claim about the other callers rests on the commit message.

Not checked: The code was not read, so the claim that the other callers of XLogPrefetcherBeginRead never have a current record is unchecked. The stability of the TAP test across branches and buildfarm animals could not be judged.

## Prior attempts

None found before 2026-10-25.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Can the bug occur with the default maintenance_io_concurrency, for example with a record that has enough block references that need I/O?
2. Should the TAP test be committed and backpatched, or kept only as a reproducer?
3. How can a user detect and repair a cluster whose timeline history file already has a wrong switch point from this bug?
4. Should a separate patch make FinishWalRecovery fail loudly when it cannot re-read the last replayed record?

## Touch points

- **WAL / replication / standby:** The fix changes xlogprefetcher.c, which runs in the startup process and decides where WAL ends at a recovery target.
- **backpatchability:** The commit message asks for a backpatch through 15, with a TAP test written to run unchanged in 15 to 18.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a new recovery TAP test, 061_recovery_target_prefetch.pl, that depends on the WAL pattern of an SP-GiST index build.

## Who would commit it, and when

- Owner: qualified committer engaged. The CommitFest entry lists Tomas Vondra as author and the commit message carries a backpatch line; the owner needs to know the WAL replay and recovery prefetcher code.
- Priority: proceed; audience niche; review cost low. The bug needs a recovery target shortly before a bulk XLOG_FPI record and mostly maintenance_io_concurrency of 7 or less, but the result is a wrong timeline switch point and replay past the target. The fix is one pointer reset in one file, and three people confirmed it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
