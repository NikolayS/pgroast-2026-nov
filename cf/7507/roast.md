# Design pre-review: pgcrypto: Fix sha-crypt rounds= values that do not fit in int (BUG #19743)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7507](https://commitfest.postgresql.org/patch/7507/); [thread](https://www.postgresql.org/message-id/flat/SN7P223MB0600215618AD31B5ACE2109EDD642%40SN7P223MB0600.NAMP223.PROD.OUTLOOK.COM), 13 of 13 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived. The only one raised, standards-compat-1, was refuted: per message #9, strtoint() already saturates on REL_18_STABLE, so the patch changes only NOTICE text and tests there. The change is a fix for a reported bug (BUG #19743) inside existing pgcrypto code, reusing the existing clamp-with-NOTICE path. Confidence is medium because only 3 of 13 messages and the v1 patch prose were read.

Not checked: The v2 to v5 patch prose and the code were not read, so the final test design, the 10-character NOTICE cap and the dependence on commit 426551c8cf49 could not be checked.

## Prior attempts

None found before 2026-10-23.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is commit 426551c8cf49 present on every branch targeted for backpatch, so that strtoint() saturates there as v4 and v5 assume?
2. Are the positive-overflow tests stable under statement_timeout on slow or heavily loaded buildfarm animals, or can they be written without a timeout?
3. Is the changed NOTICE text for in-range non-canonical tokens such as rounds=+10 and rounds=010 acceptable in a REL_18 minor release?
4. What does the final v5 commit message say about the behaviour change on master versus REL_18_STABLE?

## Touch points

- **security / privileges:** An overflowing rounds= value no longer yields a silently weaker 1000-round password hash.
- **backpatchability:** The author proposes a backpatch to REL_18_STABLE, and the NOTICE text changes for tokens such as rounds=+10.

## Who would commit it, and when

- Owner: silent. Someone who knows pgcrypto's crypt() code and the backpatch rules for contrib; neither reviewer is identified as a committer in the visible thread.
- Priority: proceed; audience niche; review cost low. The fix affects only users who pass raw sha-crypt salts with overflowing rounds= values, but it closes a silent weakening of password hashes. The patch touches one source file and one test, and a reviewer has already said v5 looks good.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
