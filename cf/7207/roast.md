# Design pre-review: Fix timeline history after recovery stops on an ancestor

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7207](https://commitfest.postgresql.org/patch/7207/); [thread](https://www.postgresql.org/message-id/flat/29641785480901%40mail.yandex.ru), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible scenario in message #1, and the new parent matches the PrevTimeLineID that the end-of-recovery record already uses. Three reviewers agree on lastRecTLI as the parent. The open points are the back-patch decision and the empty v2 commit message, and both belong to the committer.

Not checked: The code was not read, so it is not confirmed that lastRecTLI is always valid when writeTimeLineHistory() runs, or that no other path reads the old parent. The two linked discussions in message #1 were also not available.

## Prior attempts

None found before 2026-07-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which concrete tool or server path (pg_rewind, standby timeline choice, an archive tool) misbehaves today with the wrong history file, to justify a back-patch?
2. Does any code path or external tool depend on the old parent value, or on promotion restoring the target timeline's history file from the archive?
3. What should users do with clusters and archives that already hold a history file with backward switchpoints, and should the release notes say so?
4. Is there a case where recovery ends with lastRecTLI different from recoveryTargetTLI and recoveryTargetTLI is still the correct parent?
5. Can the v2 commit message be rewritten to explain the scenario, the choice of lastRecTLI over endOfLogTLI, and the test changes?

## Touch points

- **on-disk format / pg_upgrade:** The parent timeline and switchpoint written to a new timeline history file change; the file format stays the same.
- **WAL / replication / standby:** The change is in the end-of-archive-recovery path in xlog.c and affects the history file that standbys and archive tools read.
- **backpatchability:** The author asks for a back-patch, which changes history file content in minor releases without a demonstrated broken consumer.

## Who would commit it, and when

- Owner: silent. Someone who knows the end-of-recovery and timeline switch code in xlog.c; message #2 asked Heikki and Michael to comment, and neither has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. The fix is a 3.8 KB patch to one call in xlog.c plus two tests, and it affects only PITR that stops on an ancestor timeline. Reviewers agree on the design, but no committer who knows the area has replied.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
