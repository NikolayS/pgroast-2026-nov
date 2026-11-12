# Design pre-review: pg_walinspect: fix LSN validation messages and empty range handling

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7337](https://commitfest.postgresql.org/patch/7337/); [thread](https://www.postgresql.org/message-id/flat/7B8F5F12-98A5-4618-867A-904EA1334FD4%40gmail.com), 16 of 16 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patches reword error messages so they match checks that already accept equality, and change no check, SQLSTATE, catalog or format. The withdrawn v1-0002 was the only behaviour change. Confidence is medium because the code was not read and no committer has replied in the visible thread.

Not checked: The code was not read, so it is unverified that each reworded message matches its comparison and that no other site has the same mismatch. Stability of the new boundary tests in subscription.sql and 001_basic.pl was also not checked.

## Prior attempts

- [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://www.postgresql.org/message-id/flat/B611CA13-0AF0-4FD8-9A33-0CC44F3B861B@postgresql.org) (related; Rejected, 2018-08-19). "As the plan is not to backpatch, I’ll withdraw my patch." ([message](https://www.postgresql.org/message-id/50F95E32-26E4-4A47-BD66-90C8D0758692%40postgresql.org)) Different now: P5 changed a message that was not wrong, and stopped once back-patching was ruled out. Here the messages misstate the inclusive checks on master, and the commit messages already say not back-patched.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Should the CommitFest target be changed from "stable" to match the commit message, which says the fix is not back-patched?
2. Should 0002, which touches subscriptioncmds.c and psql, go in a separate commit and thread subject from the pg_walinspect fix?
3. Beyond Surya's survey of "must be less than" messages, were "must be greater than" and similar strict wordings also checked against their comparisons?

## Touch points

- **WAL / replication / standby:** The patch rewords the ALTER SUBSCRIPTION ... SKIP error in subscriptioncmds.c; the check itself does not change.
- **psql UX conventions:** ParseVariableDouble() range messages in psql/variables.c change, for example for WATCH_INTERVAL.
- **backpatchability:** The CommitFest entry targets "stable" while the commit message says not back-patched.
- **maintenance burden ("who maintains this in 10 years"):** Changed message strings need new translations and affect anyone who matches on the error text.

## Who would commit it, and when

- Owner: silent. Any committer can own the message changes; one who knows pg_walinspect or logical replication would confirm the checks are meant to be inclusive. Michael was CC'd on back-patching but has not replied.
- Priority: proceed; audience niche; review cost low. Two patches of about 6 KiB each change only message text and tests, and four reviewers call v5 ready for committer. Few users hit these boundary errors, but the cost of committing is also small.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
