# Design pre-review: support create index on virtual generated column.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5667](https://commitfest.postgresql.org/patch/5667/); [thread](https://www.postgresql.org/message-id/flat/CACJufxGao-cypdNhifHAdt8jHfK6-HX%3DtRBovBkgRuxw063GaA%40mail.gmail.com), 28 of 28 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b on already-exists-1 and prior-attempts-1. Message #9 says an expression index already gives the same result and that indexing a virtual column removes the cheap SET EXPRESSION. No message answers it, and the v9 author agreed with it in message #28. Both objections are fixable, since message #15 proposed a narrower version and did not reject the goal. The list must agree on the goal and on one design before code review makes sense.

Not checked: No code was read, and only 3 of 28 messages were visible. The level 2 objections on v1 to v7, v8 and v9 were not evaluated, because the proposal stops at level 1b.

## Prior attempts

None found before 2025-03-26.

## Objections a reviewer is likely to raise

### 1. Expression indexes on the generation expression already meet the stated need

A user who wants index support for a virtual column writes `create index on example (lower(regular_name))` today. Queries on `lowercase_name` should already use it, because the planner sees the expanded expression. The patch adds a second spelling and permanent catalog, ALTER TABLE and planner code for little gain.

Tom Lane noted in message #9 that this "does nothing you haven't been able to do for decades with expression indexes". Corey's use case in message #11 is queries that move from the expression to the virtual column and still hit the existing index. Virtual columns are expanded to their expression before index matching, so the existing expression index should already serve those queries. Message #10 shows both index forms give the same pg_index data and plans, and Zizhuan Liu accepted Tom's position in message #28.

**Question:** Which query or workload fails today with an expression index on the generation expression, and how does the patch fix it?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#9](https://www.postgresql.org/message-id/1178617.1753217685%40sss.pgh.pa.us), [#10](https://www.postgresql.org/message-id/CACJufxHT-1a-uP9pEq6rpuvq8mS0V0xOHr30R4THsDeo5DyTUA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADkLM%3Dc09mZi_XFix5EyHXPz1Z0k7%3D-DBV%2BiAOicCrDiqCqFrw%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/tencent_FCFC491B1EEDFF44DC37B3650F356BA30E06%40qq.com); `already-exists-1`</sub>

### 2. Tom Lane's objection to the goal has no answer in the thread

A user indexes a virtual generated column and later runs alter table ... set expression, expecting a cheap catalog change. Under v1 to v7 the command rebuilds the index. Under v8 the index silently keeps the old expression, and under v9 Kirill Reshke found the index corrupted. The user finds out when the command is slow or when a query returns wrong results.

Message #9 objects to the goal itself. The point of a virtual column is that set expression costs nothing, and expression indexes already give the same result. Corey Huinker's migration use case in message #11 is met by the existing expression index, because v1 to v7 store the same pg_index data (message #10). The v9 author agreed with Tom Lane in message #28, and no later message gives a reason the gain outweighs the lost property.

**Question:** Which users need an index on the virtual column itself rather than an expression index, and which set expression behaviour should they get?

<sub>fixable, level 1b, desirability; messages [#9](https://www.postgresql.org/message-id/1178617.1753217685%40sss.pgh.pa.us), [#10](https://www.postgresql.org/message-id/CACJufxHT-1a-uP9pEq6rpuvq8mS0V0xOHr30R4THsDeo5DyTUA%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CADkLM%3Dc09mZi_XFix5EyHXPz1Z0k7%3D-DBV%2BiAOicCrDiqCqFrw%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/c3123626-1ab2-48f1-a4d1-8160db5bb9e7%40eisentraut.org), [#27](https://www.postgresql.org/message-id/CALdSSPhOSSspT89VtZfTLB2xQQV5Hpu%3D90PvU1XZEuwLfOqC7w%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/tencent_FCFC491B1EEDFF44DC37B3650F356BA30E06%40qq.com); `prior-attempts-1`</sub>

## Questions for the author

1. Which query or workload does not get the needed plan today from an expression index on the generation expression?
2. Which design does the CommitFest entry now propose: v1 to v7 tracking, v8 plain expansion, or v9 stored values?
3. What does alter table ... set expression do to an indexed virtual column: rebuild the index, mark it invalid, leave it unchanged, or raise an error?
4. Why is the plan in message #15, which blocks SET EXPRESSION while the column is indexed, not enough?
5. Under v8, do pg_dump and \d show the index on the column or on the expanded expression?

## Touch points

- **on-disk format / pg_upgrade:** v9 stores virtual column values in index tuples, and v1 to v7 add a pg_index column.
- **catalog changes:** v1 to v7 add pg_index.indattrgenerated (int2vector) and document it in catalogs.sgml.
- **ABI / extension API:** v1 changes execnodes.h and relcache.h; v9 adds paired raw and expanded fields to IndexInfo, RelationData and IndexOptInfo.
- **SQL-standard conflict or non-standard syntax:** create index accepts a virtual generated column as a key column, with version-dependent meaning after SET EXPRESSION.
- **planner behaviour change:** v9 expands index expressions at plan time and changes genericcostestimate() and cost_index().
- **pg_dump / restore round-trip:** v1 changes ruleutils.c, and nobody says whether v8 dumps the column or the expanded expression.
- **psql UX conventions:** \d shows the index on the column name in v1; the v8 display is not settled.
- **maintenance burden ("who maintains this in 10 years"):** Every variant adds index handling in ALTER TABLE paths, and v1 to v7 add catalog state each such path must check.
- **performance on the hot path:** v9 changes index costing code that runs for every index, including for tables without virtual columns.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #9 says an expression index on the generation expression already covers the need.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane objected to the goal in message #9 and Peter Eisentraut proposed a narrower scope in message #15; neither supports the current patches. An owner would need to know generated columns, index DDL and ALTER TABLE.
- Priority: needs design rethink; audience niche; review cost high. The thread shows one interested user, and an expression index appears to cover his case. Three incompatible designs are on the table, they touch the catalog, ALTER TABLE and planner code, and the narrower plan in message #15 has no answer.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
