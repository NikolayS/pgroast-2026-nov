# Design pre-review: Fix the language for new CREATE SUBSCRIPTION features

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7505](https://commitfest.postgresql.org/patch/7505/); [thread](https://www.postgresql.org/message-id/flat/CAA-aLv4tAU6yY%3DOUa7Fisy%2BKk_4bS-mJreQUJjU-HhJOA9TB-w%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at any level. The patch changes documentation wording only, mostly in text new in v19, and adds no syntax, GUC, catalog or behaviour change. Peter Smith and Matemática A3K approved v2 in the thread. One accuracy check remains for the committer: the reworded retention-limit note for retain_dead_tuples.

Not checked: The material shows only the v1 hunks, so the final v2 wording could not be checked. The accuracy of the retention-limit note against server code was not verified.

## Prior attempts

None found before 2026-10-07.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the reworded retention-limit note for retain_dead_tuples match what the server does when another subscription has the limit set to 0 or not yet exceeded?
2. Does v2 change protocol.sgml as well as func-admin.sgml, given that message #7 mentions only the function documentation?
3. Which v2 hunks apply to v19 and which only to master, if any touched text does not exist in v19?

## Touch points

- **backpatchability:** Masahiko Sawada says the fixes can be backpatched to v19, where most of the touched text was added.

## Who would commit it, and when

- Owner: silent. A committer who knows the logical replication documentation, in particular the retain_dead_tuples text; the visible thread shows no committer taking the patch.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 5 KiB of wording fixes in documentation read by users of logical replication, and reviewers approved v2. It needs a committer to pick it up for v19.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
