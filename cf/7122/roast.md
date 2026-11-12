# Design pre-review: FROM clause before SELECT

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7122](https://commitfest.postgresql.org/patch/7122/); [thread](https://www.postgresql.org/message-id/flat/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Level 1b fails on three fixable objections. The main argument is psql completion, but the proposal does not say why psql cannot use the text after the cursor (already-exists-1). The only outside demand is for the v1 order, which v2 rejects (demand-1). Message #4 says completion centers on a single relation (prior-attempts-1). The goal may be fine, and the thread has one supporter, so this is a discussion to finish before code review.

Not checked: Without the code, the claim of no new grammar conflicts and the effect on parser size could not be checked. The material also does not show what the completion tests cover under the v2 order.

## Prior attempts

- [Tab completion for SELECT in psql](https://www.postgresql.org/message-id/flat/CAMyN-kB_xrU4iYdcF1j=tijgO1DSyjtb3j96O4UEj91XZrZcMg@mail.gmail.com) (related; Returned with feedback, 2018-10-01). "I think the completion needs to be smarter to be acceptable." ([message](https://www.postgresql.org/message-id/03404486-d6bf-5cb3-c937-c4590a542402%40iki.fi)) Different now: P3 completed select-list columns without knowing the tables. This proposal changes the grammar so the tables are named first and completion can use them.

## Objections a reviewer is likely to raise

### 1. A psql completion gap is being fixed with server grammar

An application developer pastes a FROM-first query from a psql session into application code. A SQL linter or proxy that embeds a Postgres-compatible parser rejects it, and the developer learns this only at deploy time. The only stated gain was select-list completion inside psql.

Message #1 and the 0002 commit message present completion as the practical argument. They say the completer sees only text left of the cursor. That is a design choice in psql's tab-complete code: readline also exposes the rest of the line, and a client such as pgcli reads the whole buffer and offers select-list columns for SELECT ... FROM t. A committer would likely ask why a client-side limit justifies a permanent server grammar extension.

**Question:** Why can psql not read the text after the cursor, for example a typed "SELECT  FROM t", to complete select-list columns without any grammar change?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru); `already-exists-1`</sub>

### 2. Demand shown is for v1 order, but v2 ships an order nobody uses

A DuckDB or ClickHouse user pastes `FROM films SELECT title WHERE len > '02:00'` into psql and gets a syntax error. The only outside users who already write FROM-first queries cannot reuse them. The v2 form is then used only by people who learn it fresh from the Postgres docs.

The only demand outside the thread is that two other systems ship the v1 placement (messages #1 and #3). Message #3 admits that "nobody ... ships the placement in v2-patch grammar", and message #4 asks for v1 for that reason. The rest of the demand is the author's teaching sessions and the stated motivation "just for fun" (message #1). No -general or -bugs reports and no user requests appear in the material.

**Question:** Which users outside this thread have asked for FROM-first queries in Postgres, and which placement do they actually write today?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru), [#4](https://www.postgresql.org/message-id/8092B28C-07B7-419F-9769-E89850D2D843%40gmail.com); `demand-1`</sub>

### 3. Completion argument may repeat the quality gap that stalled earlier select-list completion

A user writes `from orders o join customers c using (id) select ` and presses tab. psql offers columns of one table only, or bare names that are ambiguous across both. The permanent grammar extension is then in the release, but the completion that justified it falls short in the common multi-table case.

Message #1 presents 0002 as "the actual pro-argument", and the commit message calls it "the practical argument". [P3] stalled because reviewers found select-list completion too broad and too narrow, and its key sentence asked for smarter completion. Message #4 reports that the current rules are "mostly centered on a single relation" and asks for tests with joins, aliases and qualified names. The material does not show that 0002 was updated for v2 placement, so the main benefit is not yet shown for the queries users write.

Precedent: [Tab completion for SELECT in psql](https://www.postgresql.org/message-id/flat/CAMyN-kB_xrU4iYdcF1j=tijgO1DSyjtb3j96O4UEj91XZrZcMg@mail.gmail.com) (Returned with feedback, 2018-10-01)

**Question:** How does the v2 completion behave for joins, table aliases and qualified columns, and do the tests cover those cases?

<sub>fixable, level 1b, psql UX conventions; messages [#1](https://www.postgresql.org/message-id/8B32D9DE-6152-485F-AB53-9405A04C757A%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/9B4457AA-6BFD-45F5-9F24-88AA9A607EDE%40yandex-team.ru), [#4](https://www.postgresql.org/message-id/8092B28C-07B7-419F-9769-E89850D2D843%40gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. Why can psql not read the text after the cursor, for example a typed "SELECT FROM t", to complete select-list columns without a grammar change?
2. Which users outside this thread have asked for FROM-first queries in Postgres, and which clause order do they write today?
3. How does the v2 completion behave for joins, table aliases and qualified columns, and do the tests cover those cases?
4. Why should Postgres define the v2 select-list-last order, which no other system accepts, when DuckDB and ClickHouse users already write the v1 order?
5. What does the bare FROM form give users beyond TABLE and FROM ... SELECT *, given that a truncated statement would then run as SELECT *?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch adds a non-standard query form that starts with FROM, with the select list last or omitted.
- **psql UX conventions:** The patch changes tab-complete.in.c to offer FROM as a command start and to complete select-list and GROUP BY columns.
- **maintenance burden ("who maintains this in 10 years"):** gram.y gets parallel productions that each future SELECT clause must also reach, and pl_gram.y learns FROM as a statement start.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The stated gain is psql completion, which a client-side change might deliver without new server grammar.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut added himself as committer in the CommitFest entry on 2026-11-02 but has not posted in the visible thread. The owner must know gram.y and the SQL standard.
- Priority: fix design first; audience niche; review cost medium. The shown benefit is select-list completion for psql users, and message #4 says it centers on a single relation. The patch is small, but it adds permanent syntax to gram.y, and the thread has not chosen between the v1 and v2 order.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
