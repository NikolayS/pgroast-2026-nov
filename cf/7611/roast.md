# Design pre-review: psql: Fix REPACK index completion after a column list

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7611](https://commitfest.postgresql.org/patch/7611/); [thread](https://www.postgresql.org/message-id/flat/CAG7L5%2B7Xmus%2BJLa0m%3DLpw49wU%3DsecPm5dzNd2ibmfvhdSYEfqA%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible example in message #1, the fix is confined to psql tab completion for an existing command, and it adds no syntax, GUC, catalog state or server code. The remaining points are small questions about completion behaviour, not design objections. Confidence is medium because the code was not read and nobody has replied to v3.

Not checked: The code was not read, so the word-position matching in v3 could not be checked against multi-column lists, schema-qualified table names or option lists without ANALYZE. Messages #4 and #5 were not shown in full.

## Prior attempts

None found before 2026-11-08.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. With v3, does `REPACK (VERBOSE) t (a) USING INDEX <Tab>` still offer index names, although the server rejects a column list without ANALYZE?
2. What does v3 offer for `REPACK (ANALYZE) t (a) <Tab>`: the same as before the patch, or nothing?
3. Which release first shipped REPACK, and do you ask for a backpatch to it?
4. Does the table lookup still find the table when the column list has several columns or spaces, such as `(c1, c2)`?

## Touch points

- **psql UX conventions:** The patch changes which words and index names psql offers for REPACK in src/bin/psql/tab-complete.in.c.
- **backpatchability:** The CommitFest entry is tagged Bugfix, and the thread does not say which release first shipped REPACK.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds word-position rules for REPACK that must follow any later change to the REPACK grammar.

## Who would commit it, and when

- Owner: silent. A committer who knows psql tab completion and the REPACK grammar would have to own it. No committer is visible in the thread; the only reviewer is Chao Li.
- Priority: proceed; audience niche; review cost low. The fix helps psql users who type REPACK with a column list or a typed USING. The two patches are under 3 KB each, touch one psql source file and one TAP test, and add no new user-visible surface beyond completion.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
