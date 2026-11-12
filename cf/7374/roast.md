# Design pre-review: psql: avoid CREATE command completion after GRANT/REVOKE CREATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7374](https://commitfest.postgresql.org/patch/7374/); [thread](https://www.postgresql.org/message-id/flat/OSOP286MB78539E88C47403451BFFF140F18A2%40OSOP286MB7853.JPNP286.PROD.OUTLOOK.COM), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patch is a narrow psql tab-completion bug fix in one file, and the brief says a committer reviewed it and pushed v3 (message #7). The problem and the change are stated in message #1, and a reviewer confirmed the behaviour in message #2.

Not checked: The code was not read, so the exact match conditions and their effect on other inputs that end in CREATE could not be checked. Messages #4 to #7 were not shown; the push is taken from the brief and reviewer notes.

## Prior attempts

None found before 2026-10-01.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do other privilege names that are also command keywords, such as TRUNCATE, reach a generic command completion rule after GRANT or REVOKE?
2. Does the patch add tab-completion tests for the three new cases?
3. Should the commit message mention the REVOKE GRANT OPTION FOR CREATE case that v2 added?
4. Is the fix meant for back branches or for master only?

## Touch points

- **psql UX conventions:** psql now offers ON after GRANT CREATE, REVOKE CREATE and REVOKE GRANT OPTION FOR CREATE.
- **backpatchability:** The entry is tagged Bugfix, and the thread does not say whether the fix goes to back branches.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao reviewed the patch, asked for a comment change and pushed v3, per the brief and message #7.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 1 KiB in one psql file and fixes wrong suggestions for interactive users who type GRANT or REVOKE. The brief says v3 was pushed on 2026-10-07 while the CommitFest entry still says Needs review, so the entry status needs an update.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
