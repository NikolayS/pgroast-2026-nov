# Design pre-review: ON EMPTY clause for aggregate and window functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6943](https://commitfest.postgresql.org/patch/6943/); [thread](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DVS%3DfSKxfimW6Th9iu_xjbxOEAKg4eYwaa%3DSMg3X8pHaQ%40mail.gmail.com), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b. In message #11 the author accepts that v3 never differs from COALESCE, and message #7 covers the remaining case with a count()-based CASE (demand-1). The costs named in message #9 fall on everyone (maintenance-fit-1), the migration need names no database (demand-2), and the standard edition is not published (standards-compat-2). The list must agree that the clause is wanted, and in which form, before code review makes sense.

Not checked: The code was not read, and the v3 file list is not shown. Whether postgres_fdw and other code that rebuilds or deparses Aggref keep the ON EMPTY field could not be checked.

## Prior attempts

- [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (related; Rejected, 2018-07-19). "Yeah, it's a handy syntax, but it's not that much better than using a subselect or a writing a user-defined function. And there's risk of future conflicts with the SQL standard." ([message](https://www.postgresql.org/message-id/14ac5bbe-e9fb-2ce4-2d6d-698f8da29449%40iki.fi)) Different now: P9 added syntax outside the standard, while this clause comes from an expected standard edition. Both offer a new spelling for a result that an existing query form already gives.

## Objections a reviewer is likely to raise

### 1. No user need shown beyond what COALESCE already gives

A user writes sum(x, 0 ON EMPTY) and gets the same result as coalesce(sum(x), 0), with no other effect. Postgres still keeps a second spelling in gram.y, ruleutils and the docs. Future aggregate work has to keep both spellings in step.

In message #11 the author says he found no case where v3 differs from COALESCE and will no longer argue that it adds value. Tom in message #7 shows that the one case COALESCE cannot cover, an aggregate that returns NULL for non-empty input, is handled by CASE WHEN count(x) > 0. The thread reports no user who asked for this syntax. [P9] was rejected because its syntax was not much better than an existing query form.

Precedent: [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (Rejected, 2018-07-19)

**Question:** Which user, query or field report needs ON EMPTY where coalesce(agg(...), value) or a count()-based CASE is not enough?

<sub>fixable, level 1b, demand; messages [#7](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#11](https://www.postgresql.org/message-id/CAM2%2B6%3DVaUg4y5H2cpyKvuVgemL7gqdVYtv7hr1nhzJXMzX5HRQ%40mail.gmail.com); `demand-1`</sub>

### 2. General ON EMPTY adds grammar and node upkeep for COALESCE semantics

Every user pays for the larger Bison grammar on each parse, and committers maintain new Aggref and WindowFunc fields, deparse code and a catversion bump. All of this produces results that users already get from COALESCE(agg(x), v).

Message #11 says v3 gives the same answer as COALESCE in every case the author found. Message #9 names the costs: development, maintenance and some parse-speed penalty, for a small benefit. Message #7 offers a count()-based CASE for the rare aggregate that returns NULL on non-empty input. The cost falls on everyone, and the gain is a second spelling of COALESCE.

Precedent: [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (Rejected, 2018-07-19)

**Question:** Which real migration or user case needs the ON EMPTY spelling instead of COALESCE, and how often does it occur?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#7](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#9](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us), [#11](https://www.postgresql.org/message-id/CAM2%2B6%3DVaUg4y5H2cpyKvuVgemL7gqdVYtv7hr1nhzJXMzX5HRQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Migration demand is asserted, with no source system or users named

Postgres adds the grammar rule to help people port queries from other databases. If none of the usual source systems uses this spelling, nobody ports such a query. Everyone still pays the extra grammar and parsing cost Tom describes in message #9.

The only demand in the thread is in message #8, where Vik wants the syntax to help conversion from databases that have it. No database is named, and no migration case is shown. The standard text is quoted second-hand, and the edition is not yet published. Vik says the syntax may still change there. Message #9 says the cost falls on every parse, while the gain goes only to those who write the clause.

**Question:** Which shipping database systems accept agg(x, v ON EMPTY), and is there a migration report where its absence blocked or complicated a port to Postgres?

<sub>fixable, level 1b, demand; messages [#8](https://www.postgresql.org/message-id/e9454ec0-6b35-49d6-aac3-7c5f1deeb309%40postgresfriends.org), [#9](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us); `demand-2`</sub>

### 4. Adopts draft syntax before the standard edition is published

Postgres 20 ships agg(x, d ON EMPTY). The published edition changes the spelling, for example to match JSON_VALUE's DEFAULT expr ON EMPTY. Postgres then carries two spellings, or users who adopted the early form find that their SQL does not port.

Message #8 says the next edition is not out and invites a better proposal for the syntax. Tom (message #7) and Robert (message #6) find the current spelling confusing because the substitute value looks like an aggregate argument. The same clause in SQL/JSON, which Postgres already implements, requires DEFAULT before the expression. COALESCE already gives the same result, so the risk of a future conflict is weighed against little gain, as in [P9].

Precedent: [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (Rejected, 2018-07-19)

**Question:** Is the ON EMPTY spelling for aggregates final in the standard draft, and why not wait for the published edition before taking it into the grammar?

<sub>fixable, level 1b, SQL-standard conflict or non-standard syntax; messages [#6](https://www.postgresql.org/message-id/CA%2BTgmoYfZW9y_T8A1nykRUcvJ2FQtMpBjgkq%3D%2BTCdq0-odF4%3Dw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/636820.1789651904%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/e9454ec0-6b35-49d6-aac3-7c5f1deeb309%40postgresfriends.org), [#9](https://www.postgresql.org/message-id/645015.1789660502%40sss.pgh.pa.us); `standards-compat-2`</sub>

## Questions for the author

1. Which user query, field report or migration case needs ON EMPTY where coalesce(agg(...), value) or a count()-based CASE is not enough?
2. Which shipping database systems accept agg(x, v ON EMPTY), and is there a report where its absence complicated a port to Postgres?
3. Is the ON EMPTY spelling for aggregates final in the standard draft, and why not wait for the published edition before adding it to the grammar?
4. Which version is proposed for commit, the general v3 or the WIP patch limited to sum() and product()?
5. For aggregates other than sum() and product(), why would a later standard edition define ON EMPTY with COALESCE semantics and not the v1 empty-input rule?

## Touch points

- **catalog changes:** v1 to v3 add fields to Aggref and WindowFunc that are stored in pg_rewrite, which needs a catversion bump.
- **ABI / extension API:** New fields in Aggref, WindowFunc and FuncCall affect extensions that walk, copy or deparse these nodes.
- **SQL-standard conflict or non-standard syntax:** New agg(args, value ON EMPTY) syntax taken from an unpublished standard edition, and extended beyond sum() and product().
- **planner behaviour change:** v1 marks ON EMPTY aggregates non-partial, and message #10 excludes them from the MIN/MAX index path.
- **pg_dump / restore round-trip:** The WIP patch deparses views as COALESCE, while v1 to v3 add ON EMPTY deparse in ruleutils.c.
- **maintenance burden ("who maintains this in 10 years"):** A second spelling of COALESCE must be kept in gram.y, node fields, ruleutils.c and every code path that rebuilds an Aggref.
- **performance on the hot path:** Message #9 names a parse-speed cost from the larger grammar, and v1 adds an executor expression step with JIT support.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** COALESCE(agg(x), v) and a count()-based CASE already give every result the proposal offers.

## Who would commit it, and when

- Owner: qualified committer engaged. A committer is registered on the CommitFest entry but has not answered the open objections. Tom Lane and Robert Haas take part in the thread, and both object.
- Not now, show demand first: A published standard edition that fixes this spelling and its semantics, or named database systems that accept agg(x, v ON EMPTY) together with a migration report where its absence was a problem. A user case that COALESCE and a count()-based CASE cannot express would also change the answer.
- Priority: park; audience unclear; review cost medium. The thread names no user or source database that needs the clause, and the standard edition it follows is not published. The general version touches grammar, node fields, planner, executor and deparse for results COALESCE already gives, so waiting for the published text costs little.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
