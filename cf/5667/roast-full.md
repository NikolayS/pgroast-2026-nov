# Design pre-review: support create index on virtual generated column.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5667](https://commitfest.postgresql.org/patch/5667/) (Needs review); authors: Jian He (stayreal)
- Thread: [support create index on virtual generated column.](https://www.postgresql.org/message-id/flat/CACJufxGao-cypdNhifHAdt8jHfK6-HX%3DtRBovBkgRuxw063GaA%40mail.gmail.com), started 2025-03-26
- Messages read: 28 of 28
- Patch prose read: v8-0001-index-on-virtual-generated-column.patch, v1-0001-support-create-index-on-virtual-generated-column.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b on already-exists-1 and prior-attempts-1. Message #9 says an expression index already gives the same result and that indexing a virtual column removes the cheap SET EXPRESSION. No message answers it, and the v9 author agreed with it in message #28. Both objections are fixable, since message #15 proposed a narrower version and did not reject the goal. The list must agree on the goal and on one design before code review makes sense.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, and only 3 of 28 messages were visible. The level 2 objections on v1 to v7, v8 and v9 were not evaluated, because the proposal stops at level 1b.

## The proposal, restated

**Problem.** Postgres rejects indexes on virtual generated columns, so users must write an expression index with the generation expression instead. Corey Huinker wants queries that move from the expression to the virtual column to keep using the existing index. Tom Lane replied that expression indexes already cover this need.

**Proposed design.** Jian He's v1 rewrites an index on a virtual generated column into an expression index on the generation expression. It records the source column in a new pg_index column so SET EXPRESSION and SET DATA TYPE rebuild the index. His v8 drops that tracking and just expands the expression at create index, while Zizhuan Liu's v2 to v9 store column values in the index and expand them at plan time.

**User-visible surface.**
- create index accepts a virtual generated column as a key column; v9 also allows it in expressions, INCLUDE and where predicates.
- New pg_index column indattrgenerated (int2vector) lists the virtual generated columns an index uses (v1 to v7 only).
- alter table ... alter column ... set expression rebuilds dependent indexes (v1 to v7), or leaves them unchanged (v8).
- Unique and deferrable unique constraints on virtual generated columns (v9 only).

**What the author claims.**
- An index on a virtual generated column stores the same pg_index data as the matching expression index, so the planner produces the same plans.
- v1 to v7 support all index types: btree, gist, spgist, hash, gin and brin.
- v8 is much simpler but must document that SET EXPRESSION does not affect existing indexes.
- The DefineIndex() comment saying RelationGetIndexExpressions() needs changes is wrong.
- v9 matches stored generated columns in most catalog behavior and passes the regression tests.

**Thread state.** Kirill Reshke found that v9 corrupts the index after SET EXPRESSION and crashes on a deferred unique constraint, and v9 no longer applies. Zizhuan Liu then said he agrees with Tom Lane that the feature should not be done, and no newer patch or reply follows.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-03-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Expression indexes on the generation expression already meet the stated need

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user who wants index support for a virtual column writes `create index on example (lower(regular_name))` today. Queries on `lowercase_name` should already use it, because the planner sees the expanded expression. The patch adds a second spelling and permanent catalog, ALTER TABLE and planner code for little gain.

Tom Lane noted in message #9 that this "does nothing you haven't been able to do for decades with expression indexes". Corey's use case in message #11 is queries that move from the expression to the virtual column and still hit the existing index. Virtual columns are expanded to their expression before index matching, so the existing expression index should already serve those queries. Message #10 shows both index forms give the same pg_index data and plans, and Zizhuan Liu accepted Tom's position in message #28.

Thread: [#9 (Tom Lane, 2025-07-22)](https://www.postgresql.org/message-id/1178617.1753217685%40sss.pgh.pa.us), [#10 (jian he, 2025-07-24)](https://www.postgresql.org/message-id/CACJufxHT-1a-uP9pEq6rpuvq8mS0V0xOHr30R4THsDeo5DyTUA%40mail.gmail.com), [#11 (Corey Huinker, 2025-07-30)](https://www.postgresql.org/message-id/CADkLM%3Dc09mZi_XFix5EyHXPz1Z0k7%3D-DBV%2BiAOicCrDiqCqFrw%40mail.gmail.com), [#28 (ZizhuanLiu X-MAN, 2026-11-06)](https://www.postgresql.org/message-id/tencent_FCFC491B1EEDFF44DC37B3650F356BA30E06%40qq.com)

**Question:** Which query or workload fails today with an expression index on the generation expression, and how does the patch fix it?

Counter-check: held. Postgres expands virtual generated columns before index matching, so an expression index already serves the queries in message #11. Message #10 shows the same pg_index data and plans, and the thread shows no case that the expression index cannot serve.

### 2. Tom Lane's objection to the goal has no answer in the thread

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 2 · desirability, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A user indexes a virtual generated column and later runs alter table ... set expression, expecting a cheap catalog change. Under v1 to v7 the command rebuilds the index. Under v8 the index silently keeps the old expression, and under v9 Kirill Reshke found the index corrupted. The user finds out when the command is slow or when a query returns wrong results.

Message #9 objects to the goal itself. The point of a virtual column is that set expression costs nothing, and expression indexes already give the same result. Corey Huinker's migration use case in message #11 is met by the existing expression index, because v1 to v7 store the same pg_index data (message #10). The v9 author agreed with Tom Lane in message #28, and no later message gives a reason the gain outweighs the lost property.

Thread: [#9 (Tom Lane, 2025-07-22)](https://www.postgresql.org/message-id/1178617.1753217685%40sss.pgh.pa.us), [#10 (jian he, 2025-07-24)](https://www.postgresql.org/message-id/CACJufxHT-1a-uP9pEq6rpuvq8mS0V0xOHr30R4THsDeo5DyTUA%40mail.gmail.com), [#11 (Corey Huinker, 2025-07-30)](https://www.postgresql.org/message-id/CADkLM%3Dc09mZi_XFix5EyHXPz1Z0k7%3D-DBV%2BiAOicCrDiqCqFrw%40mail.gmail.com), [#15 (Peter Eisentraut, 2026-03-13)](https://www.postgresql.org/message-id/c3123626-1ab2-48f1-a4d1-8160db5bb9e7%40eisentraut.org), [#27 (Kirill Reshke, 2026-11-05)](https://www.postgresql.org/message-id/CALdSSPhOSSspT89VtZfTLB2xQQV5Hpu%3D90PvU1XZEuwLfOqC7w%40mail.gmail.com), [#28 (ZizhuanLiu X-MAN, 2026-11-06)](https://www.postgresql.org/message-id/tencent_FCFC491B1EEDFF44DC37B3650F356BA30E06%40qq.com)

**Question:** Which users need an index on the virtual column itself rather than an expression index, and which set expression behaviour should they get?

Counter-check: held. No message answers message #9, and message #28 agrees with it. Peter Eisentraut proposed a narrower version in message #15 and did not reject the goal, so a new version could answer this.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | v9 stores virtual column values in index tuples, and v1 to v7 add a pg_index column. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | v1 to v7 add pg_index.indattrgenerated (int2vector) and document it in catalogs.sgml. |
| ABI / extension API | **yes** | v1 changes execnodes.h and relcache.h; v9 adds paired raw and expanded fields to IndexInfo, RelationData and IndexOptInfo. |
| SQL-standard conflict or non-standard syntax | **yes** | create index accepts a virtual generated column as a key column, with version-dependent meaning after SET EXPRESSION. |
| new GUC | no |  |
| planner behaviour change | **yes** | v9 expands index expressions at plan time and changes genericcostestimate() and cost_index(). |
| pg_dump / restore round-trip | **yes** | v1 changes ruleutils.c, and nobody says whether v8 dumps the column or the expanded expression. |
| psql UX conventions | **yes** | \d shows the index on the column name in v1; the v8 display is not settled. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every variant adds index handling in ALTER TABLE paths, and v1 to v7 add catalog state each such path must check. |
| performance on the hot path | **yes** | v9 changes index costing code that runs for every index, including for tables without virtual columns. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #9 says an expression index on the generation expression already covers the need. |

## 5. Questions the author must answer before anyone reviews code

1. Which query or workload does not get the needed plan today from an expression index on the generation expression?
2. Which design does the CommitFest entry now propose: v1 to v7 tracking, v8 plain expansion, or v9 stored values?
3. What does alter table ... set expression do to an indexed virtual column: rebuild the index, mark it invalid, leave it unchanged, or raise an error?
4. Why is the plan in message #15, which blocks SET EXPRESSION while the column is indexed, not enough?
5. Under v8, do pg_dump and \d show the index on the column or on the expanded expression?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane objected to the goal in message #9 and Peter Eisentraut proposed a narrower scope in message #15; neither supports the current patches. An owner would need to know generated columns, index DDL and ALTER TABLE.
- **Liability:** The owner signs for the SET EXPRESSION and SET DATA TYPE behaviour of indexed virtual columns for good. Under v1 to v7 that includes a pg_index column every later ALTER path must check.
- **Author follow-up:** Jian He posted v1 to v8 and answered review, but no patch or reply follows message #28. The entry has needed a rebase since 2026-10-10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The thread shows one interested user, and an expression index appears to cover his case. Three incompatible designs are on the table, they touch the catalog, ALTER TABLE and planner code, and the narrower plan in message #15 has no answer.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Under v8, an index created on column b silently stops tracking b. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Indexing a virtual column changes what SET EXPRESSION costs. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Index with stored virtual column values breaks after SET EXPRESSION. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) v9 needs raw and expanded copies of every index expression. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Tracking design adds catalog state that every ALTER path must consult. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `replication-ondisk-2` New pg_index column adds dependency state every index path must follow. Refuted (not specific): Duplicate of maintenance-fit-2, which makes the same point about indattrgenerated and adds the drop column evidence from message #2.
- `maintenance-fit-3` Indexing virtual columns gives up cheap SET EXPRESSION and adds little. Refuted (not specific): Duplicate of prior-attempts-1 and already-exists-1. The added claim that the chosen semantics can never change has no support in the material.
- `already-exists-2` The v8 form duplicates expression indexes and keeps the old expression. Refuted (not specific): Duplicate of standards-compat-1, which describes the same v8 behaviour after SET EXPRESSION and the same unanswered alternative from message #15.
- `demand-1` The one stated use case already works with an expression index. Refuted (not specific): Duplicate of already-exists-1, which makes the same point from messages #9, #10 and #11.
- `demand-2` The gain is syntax, and it removes the cheap set expression. Refuted (not specific): Duplicate of prior-attempts-1. Item 4 also fits poorly, because only users who index the virtual column pay the SET EXPRESSION cost.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches. [P2] is the original generated columns patch, returned for bugs in the base feature, and that reason does not apply here. Message #18 links to earlier threads and CommitFest entries, and message #25 mentions CF6749, but the material gives no recorded outcome for them.
- standards-compat: 2 objection(s). Not applicable; two objections raised. v9's unique and deferrable constraints were not judged separately, because its reviewer withdrew support in message #28.
- replication-ondisk: 2 objection(s). v8 stores only a plain expression index and adds no persistent state beyond what expression indexes have today, so this angle finds nothing in it.
- security: 0 objection(s). No security or privilege objection at the level of the idea or design. Every variant ends up with an index whose expressions are built from the generation expression. Postgres already evaluates such expressions in expression indexes, under the same owner and security-restricted rules during REINDEX, VACUUM and ANALYZE. The deferred unique constraint crash in v9 (message #27) is an implementation defect, and the v9 author has stepped back from that version (message #28).
- maintenance-fit: 3 objection(s). Committability: no committer supports the feature in the thread. Tom Lane is against it and Peter Eisentraut proposed a narrower scope. The v9 reviewer withdrew support in message #28.
- already-exists: 2 objection(s). Two objections. The precedent [P2] is about generated columns themselves and does not apply.
- demand: 2 objection(s). Two objections are raised. The material shows one interested user, and his case looks already covered by expression indexes.

### Precedents retrieved

- P1: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P2: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - used in the review; reason summarised from its thread
- P3: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P4: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P5: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P6: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P7: [doc: virtual envs with Pl/Python](https://commitfest.postgresql.org/patch/5308/) (cf, Withdrawn, 2025-02-10) - judged not relevant; reason summarised from its thread
- P8: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant; reason summarised from its thread
- P11: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P12: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P13: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P14: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P15: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P16: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant
- P17: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P18: [Revert "Prevent incorrect updates of pg_index while reindexing pg_index itself."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c096d19b74a637443109e528000342e896b150f3) (revert, Reverted, 2011-04-19) - judged not relevant
- P19: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P21: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P22: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P23: [Add IMMUTABLE column attribute](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant

### Warnings

- CommitFest entry has 3 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/CACJufxGao-cypdNhifHAdt8jHfK6-HX%3DtRBovBkgRuxw063GaA%40mail.gmail.com). Others: https://www.postgresql.org/message-id/flat/CAMbWs4811nG3w_3sLxps%2BEAuUsffA_83ZQ-1acEH_jQeq117mg%40mail.gmail.com, https://www.postgresql.org/message-id/flat/tencent_54EF3ECB626AD14621788C36625A8109A207%40qq.com

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 181 s; cost reported by the CLI: $2.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 1 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 112 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:35:05.984Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
