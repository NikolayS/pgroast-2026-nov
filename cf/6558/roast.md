# Design pre-review: Pgbench: remove synchronous prepare

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6558](https://commitfest.postgresql.org/patch/6558/); [thread](https://www.postgresql.org/message-id/flat/2da56b59-acb6-4ee3-af6b-6e800115dc7f%40postgrespro.ru), 15 of 15 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2, because the panel raised none. The problem is stated with a reproducer in message #1, and a second case without a pooler is reported in message #14. v5 stays inside pgbench.c and its TAP test and uses the existing PQsendPrepare(), which is what message #9 asked for. Confidence is medium because nobody has reviewed v5 in the visible thread and the code was not read.

Not checked: Without the code, the panel could not check that the new state has no blocking PQgetResult() left, handles connection loss, and keeps failure counting and --report-per-command timing unchanged.

## Prior attempts

None found before 2026-01-27.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does any blocking PQgetResult() call remain in the new prepare path, including when the connection fails between results?
2. Do retry and failure counts under --max-tries and --failures-detailed stay the same when a Prepare fails in the new state?
3. Is there a plan or a tracking item to give prepareCommandsInPipeline() the same asynchronous treatment?
4. Is the documentation change from message #2 still needed, given that v5 touches only pgbench.c and the test file?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** v5 adds the CSTATE_WAIT_PREPARE_RESULT state to the pgbench state machine and leaves the pipeline path synchronous.

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas took part in the thread and asked for PQsendPrepare(), but he has not commented on v5. The owner needs to know the pgbench client state machine and libpq asynchronous result handling.
- Priority: proceed; audience niche; review cost low. The fix helps pgbench users who run -M prepared with fewer threads than clients, behind a pooler or under lock waits. v5 is a 14.7 KB patch to pgbench.c and one TAP test with no libpq change, and it waits for a review of v5.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
