# Design pre-review: Backend crash in expanded_record_info function

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7618](https://commitfest.postgresql.org/patch/7618/); [thread](https://www.postgresql.org/message-id/flat/CAHewXNnDBGifxSqHvt1Xg4XmTL4%3DkQauQRo2kyWP4Yhz0bPG9w%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. Message #1 states the problem with a reproducible crash and a backtrace, and the patch changes only three test functions in src/test/regress/regress.c. The confidence is medium because the thread has one message and nobody has read the code here.

Not checked: Without the code, nobody checked that the row-type check accepts every argument the existing tests pass, including domains over composite types. The error message is also unseen.

## Prior attempts

None found before 2026-11-10.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do the back branches contain the same three functions, and should the fix go there?
2. What error message and error code does a non-row argument produce?
3. Do other polymorphic test functions in regress.c convert their argument without a type check?
4. Does the patch add a regression test for the new check, or is the check alone intended?

## Touch points

- **backpatchability:** The entry is tagged Bugfix, and the proposal does not say whether back branches have the same three functions and need the fix.

## Who would commit it, and when

- Owner: silent. A committer who knows the expanded-record code or the regression test support functions in regress.c would have to own it. Nobody has replied to message #1.
- Priority: proceed; audience niche; review cost low. The patch is 2.6 KB and touches only src/test/regress/regress.c. It helps developers and fuzzers such as SQLsmith that run queries in the regression database, and it adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
