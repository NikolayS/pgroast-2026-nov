# Design pre-review: [Patch] Fix pg_get_multixact_stats() over-reporting members on a hot standby

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7324](https://commitfest.postgresql.org/patch/7324/); [thread](https://www.postgresql.org/message-id/flat/CA%2BQeY%2BDcnsZRL4GZD8JyJu5HD-d6p8_9M5kT4jgF3gq3Rh8n4w%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated and reproduced in message #1, and the thread agreed in messages #2 and #3 to raise an error during recovery, which copies the existing xlogfuncs.c practice. The function is new in 19, so no released behaviour changes. Confidence is medium because v2, which carries the agreed design, is not attached.

Not checked: v2 is not attached, so its error check, documentation and tests could not be judged. Whether v1's redo-path update of oldestOffset remains in the final patch, and whether it is safe during replay, also needs the code.

## Prior attempts

None found before 2026-09-21.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Can you post v2, so reviewers can see the exact error behaviour of pg_get_multixact_stats() during recovery?
2. Does v2 keep v1's update of oldestOffset while replaying XLOG_MULTIXACT_TRUNCATE_ID, or does it drop that change?
3. Will the documentation of pg_get_multixact_stats() state that the function raises an error during recovery?
4. Does v2 add a test that calls pg_get_multixact_stats() on a standby and checks the error?
5. Is the loss of the correct num_mxids and oldest_multixact on a standby acceptable, given that only num_members and members_size are wrong today?

## Touch points

- **WAL / replication / standby:** v1 changes replay of XLOG_MULTIXACT_TRUNCATE_ID in multixact.c, and the agreed v2 changes the function's behaviour on a standby.
- **backpatchability:** The entry is a bug fix targeted at 19, where pg_get_multixact_stats() is new and still in beta.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas and Michael Paquier replied in the thread, and Michael said he would make the function raise an error during recovery.
- Priority: proceed; audience niche; review cost low. The fix corrects wrong output from a function that is new in 19, so it is best settled before release. The patch touches one file, the agreed design adds no new state or strings, and a committer has offered to make the change.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
