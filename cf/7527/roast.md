# Design pre-review: BUG #19737: Empty `JSON_OBJECT` cannot use documented `ON NULL` or unique-key clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7527](https://commitfest.postgresql.org/patch/7527/); [thread](https://www.postgresql.org/message-id/flat/19737-b44cf2e08708b613%40postgresql.org), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The doc mismatch is real, but the thread shows no query that needs ON NULL or UNIQUE with an empty list (demand-1). Message #1 itself says the clauses do not change the result, and JSON_OBJECT() already returns {}. The docs-only fix from message #2 matches the current parser and was never weighed against the grammar change. The list should choose between the two fixes before code review.

Not checked: Without the code, nobody can confirm that the new gram.y productions add no conflicts, or how a view that uses the new forms is deparsed. The v3 documentation text on the difference from the standard is not shown.

## Prior attempts

None found before 2026-10-02.

## Objections a reviewer is likely to raise

### 1. No use case shown for clauses that cannot change the result

Postgres adds grammar for JSON_OBJECT(ABSENT ON NULL) and similar forms, which always return {} or []. If no user needs them, the project still carries syntax that the SQL:2016 grammar does not allow. A later standard revision or other systems may define these forms differently, and Postgres must then keep its own behavior or break callers.

The only demand in the thread is the doc mismatch in message #1, and message #1 itself says the clauses do not change the result. Anyone who hits the error can drop the clause, since JSON_OBJECT() and JSON_ARRAY() already work (message #3, message #7). Message #2 offers a docs-only fix that matches the standard, and nobody weighed it against the grammar change. Message #9 calls the grammar change "probably" right only because the docs already differ from the standard.

Precedent: [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (Rejected, 2018-07-19)

**Question:** Which real queries or SQL generators emit ON NULL or UNIQUE clauses with an empty list, and why is dropping the clause or fixing the docs not enough for them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/19737-b44cf2e08708b613%40postgresql.org), [#2](https://www.postgresql.org/message-id/CACJufxERBd7_cN9-3FSAg4%2BGEdvXcaNQBoOBr1ZQ7Kgcwx7WOQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAFjuD9ehcFRN4M1QkJDQq0Asr8v%2B8dVWcGPc5Z7ENEpy-iA5fg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAFjuD9fKAs3%3DSuywg%3D3xMM1GxLjGbCuZ8nt9GK4PFD4JbmY5ow%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAH5HC97O5fkwYkyZr-YtzugAniNNJ3YxS%3DznSw2oCeQqOB0KXQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which real queries or SQL generators emit ON NULL or UNIQUE clauses with an empty key/value list, and why is dropping the clause not enough for them?
2. Why is the grammar change better than the docs-only fix from message #2, which matches both the current parser and the SQL:2016 grammar?
3. Is the grammar change meant for master only, or for back-patching to the supported branches?
4. If it is back-patched, what happens to SQL and dumps that use the new forms on an older minor release?
5. Do JSON_OBJECTAGG and JSON_ARRAYAGG have the same gap between the documented synopsis and the parser?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch accepts empty JSON_OBJECT and JSON_ARRAY forms with ON NULL and UNIQUE clauses that the SQL:2016 grammar does not allow.
- **pg_dump / restore round-trip:** A view written with the new forms may dump as text that an older minor release cannot parse, if the fix is back-patched.
- **backpatchability:** The entry is tagged Bugfix against 18.6, and the thread does not say whether the grammar change is for master only.
- **maintenance burden ("who maintains this in 10 years"):** New gram.y productions for the empty forms must stay conflict-free and stay accepted in every later release.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** JSON_OBJECT() and JSON_ARRAY() without the clauses already return the same empty object or array.

## Who would commit it, and when

- Owner: silent. A committer who knows the SQL/JSON constructor grammar in gram.y and the SQL/JSON standard text would have to own it. None has replied in the visible thread.
- Not now, show demand first: A real query, ORM or SQL generator that emits these clauses with an empty list, or documentation from other database systems showing they accept the empty forms. The LLM search in message #5 does not count as this evidence.
- Priority: needs design rethink; audience niche; review cost low. The patch touches only gram.y, the regression tests and func-json.sgml, and the new forms return what JSON_OBJECT() and JSON_ARRAY() already return. The thread first needs to choose between this grammar change and the docs-only fix from message #2.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
