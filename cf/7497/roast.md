# Design pre-review: pg_stat_statements: fix SQL/JSON behavior keywords and JSON_TABLE query jumbling

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7497](https://commitfest.postgresql.org/patch/7497/); [thread](https://www.postgresql.org/message-id/flat/457969A9-5D24-422F-B09E-F388053DE028%40localhost.localdomain), 9 of 9 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with examples, was reported on pgsql-bugs, and the remaining patch reuses the existing query_jumble_ignore node attributes. Michael Paquier committed the tests and the behavior keyword fix and limited the query ID change to master. Confidence is medium because only 3 of 9 messages are visible and no reply to v6 is shown.

Not checked: Without the code, it could not be checked that every jumbled field of the JSON_TABLE plan nodes affects the result, or that the new NESTED PATH Const locations match the literals. The discussion in messages #4 to #9 was not visible.

## Prior attempts

None found before 2026-10-19.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do reviewers accept jumbling TableFunc.plan with the fields listed in message #1, which Sami Imseih earlier found hard to get comfortable with?
2. Is it accepted that COLUMNS (a int) and COLUMNS (b int) now share one entry, although they read different keys?
3. Should "a jsonb PATH" and "a jsonb FORMAT JSON PATH" keep separate query IDs, and which reviewer has agreed to that?
4. Were tests for WITHOUT UNIQUE and JSON_ARRAYAGG ORDER BY, which Michael Paquier listed, added to sqljson.sql?

## Touch points

- **ABI / extension API:** The patch changes jumble attributes of TableFunc, JsonExpr and JsonTablePath in primnodes.h, so query IDs seen by extensions change.
- **backpatchability:** Query IDs of JSON_TABLE and other SQL/JSON query functions change, so Michael Paquier ruled out a back-patch in message #2.
- **maintenance burden ("who maintains this in 10 years"):** Future fields added to JsonTablePathScan or JsonTableSiblingJoin are jumbled by default and need a decision each time.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier, who knows query jumbling and pg_stat_statements, reviewed the series and committed the tests and the behavior keyword fix.
- Priority: proceed; audience niche; review cost low. Only users who monitor JSON_TABLE queries with pg_stat_statements benefit, but their entries are wrong today. The remaining patch is about 20 KiB, mostly expected test output, touches two source files, and a committer who knows the area is engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
