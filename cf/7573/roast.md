# Design pre-review: Fix pg_combinebackup with truncated intermediate incremental backups

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7573](https://commitfest.postgresql.org/patch/7573/); [thread](https://www.postgresql.org/message-id/flat/CAJoZ81s07r3akkKp-nJ8sgQNoCM0nu24g3HZ0FxTSY1OqsgurQ%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The thread states the problem with a bug report and reproducer (BUG #19776), the reviewer in message #3 confirmed the failure on master and the pass with v2, and the fix adds no user-visible surface. The remaining gaps are about tests and branch choice, which a committer can settle.

Not checked: The code was not read, so the logic of the running bound and of the full_copy_possible line rests on the reviewer's account in message #3. The segment-boundary case on --with-segsize-blocks builds has no test.

## Prior attempts

None found before 2026-10-28.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches need this fix, and does the patch apply to each of them?
2. Can the test be made to fail without the fix on a --with-segsize-blocks=6 build, for example with a truncation point inside the re-extended segment?
3. Can any test, perhaps one that builds backup files by hand, cover the full_copy_possible = false line?
4. Does the pg_combinebackup TAP suite pass on v3, given that the reported pass result is from v2?
5. What should users do who already restored from an affected chain and may have stale rows?

## Touch points

- **on-disk format / pg_upgrade:** The fix changes which blocks pg_combinebackup writes into the reconstructed relation files; the incremental file format stays the same.
- **backpatchability:** This is a data-correctness bug fix, and the thread does not say which branches get it.

## Who would commit it, and when

- Owner: silent. Someone who knows the incremental backup and pg_combinebackup reconstruction code. Fujii Masao is listed as a CommitFest reviewer but has not posted in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The bug restores stale rows for users of incremental backup chains, and the fix touches one source file and one test file with no new user-visible surface. A reviewer already confirmed the failure and the fix on master.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
