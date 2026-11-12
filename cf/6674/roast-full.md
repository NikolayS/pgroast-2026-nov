# Design pre-review: Adding a stored generated column without long-lived locks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6674](https://commitfest.postgresql.org/patch/6674/) (Needs review); authors: Alberto Piai (albe)
- Thread: [Adding a stored generated column without long-lived locks](https://www.postgresql.org/message-id/flat/abkrpUwlGngF4e-d%40phidippus.sen.work), started 2026-03-17
- Messages read: 42 of 42
- Patch prose read: v12-0001-Add-SQL-level-datum-equality-tests.patch, v12-0002-Support-changing-a-column-into-a-stored-generate.patch, v1-0001-Support-changing-a-column-into-a-stored-generated.patch, v1-0002-Try-to-avoid-a-rewrite-when-adding-a-stored-gener.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread and no objection to it survived. The design is not settled. replication-ondisk-1 describes a path where clearing attmissingval without a rewrite turns existing values into NULL. standards-compat-2 shows the required constraint shape depends on an uncommitted function, with the fallback left open in message #42. The proof rule has changed three times, so the list should agree on it before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code I could not check how v12 handles atthasmissing, which lock the command takes, or whether the eligibility checks are shared with CREATE TABLE. Only 3 of the 42 messages were visible, so later messages are known only through the objections.

## The proposal, restated

**Problem.** Adding a stored generated column to a large table today rewrites the table under an AccessExclusiveLock. The author hit this in production and wants a way for operators to do this migration online.

**Proposed design.** A new ALTER TABLE subcommand turns a regular column into a stored generated column, taking the generation expression from a named, valid check constraint and never rewriting or scanning the table. The user first adds the column, a trigger, and a NOT VALID constraint, backfills, then runs VALIDATE. v12 requires the constraint to be CHECK (pg_datum_image_equal(col, expr)), a new function from a separate patch that compares binary images.

**User-visible surface.**
- New syntax: alter table ... alter column col add generated using constraint constr_name stored.
- New SQL function pg_datum_image_equal(anyelement, anyelement) returns boolean, from Matthias van de Meent's separate patch, included as v12-0001.
- The command fails on identity, serial, already generated, partition key, foreign key with actions, and foreign table columns.
- On partitioned or inheritance tables the command works only on the root without ONLY.
- psql tab completion for the new syntax, and a new injection point that detects table scans and rewrites.

**What the author claims.**
- The command fills a gap next to SET EXPRESSION and DROP EXPRESSION, as the reverse of DROP EXPRESSION.
- With a valid pg_datum_image_equal constraint, stored values match what the expression would produce, so later rewrites and pg_dump/pg_restore do not fail.
- Both = and IS NOT DISTINCT FROM are unsound: NULL results and values equal by = but with different images, like numeric 1.0 and 1.00, break consistency.
- A reviewer saw the relation file stay unchanged on a 1,000,000-row table.
- Logical replication works if the subscriber is prepared with triggers when generated columns are not published.

**Thread state.** Laurenz Albe reviewed through v10 and marked it Ready for Committer, but the author withdrew that after finding the IS NOT DISTINCT FROM flaw; Álvaro Herrera questioned whether it matters and Matthias van de Meent said it does. v12 now depends on the uncommitted pg_datum_image_equal patch, and no one has replied to it yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Clearing attmissingval can turn unrewritten tuples to NULL

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, catalog changes · `replication-ondisk-1`

An operator runs `alter table t add column c int not null default 0`, which uses a fast default with no rewrite. The backfill updates only rows `where c is distinct from expr`, VALIDATE passes, and the conversion clears attmissingval. Old tuples whose expr is 0 were never updated, so they now read NULL in a not-null column.

Message #42 says v12 clears atthasmissing and attmissingval for consistency with CREATE TABLE, and the command does no scan or rewrite. For tuples written before the ADD COLUMN, attmissingval is the only source of the column value. The proposal does not say why every tuple must physically hold the column before the catalog entry is removed. A batched, idempotent backfill that skips unchanged rows is a common pattern.

Thread: [#42 (Alberto Piai, 2026-11-06)](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com)

**Question:** What guarantees that no tuple still depends on attmissingval when it is cleared, or does the command refuse when atthasmissing is set?

Counter-check: held. SET EXPRESSION and ALTER COLUMN TYPE clear the missing value because they rewrite the table, and this command does not rewrite. Message #42 does not say what guarantees that no tuple still reads its value from attmissingval.

### 2. DDL contract depends on an uncommitted function recognized by shape

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Operators write migration runbooks against the documented form CHECK (pg_datum_image_equal(col, expr)). If the separate patch lands under another name, with other NULL semantics, or does not land, the documented form changes. In the ROW()::record *= fallback, every runbook written against the earlier form then fails with "not of the form".

In v12 the user-facing contract of the command is a literal function call inside a CHECK constraint, and that function comes from v12-0001, which has its own review (message #42). In message #42 the fallback is left open. Existing proof shortcuts such as SET NOT NULL and ATTACH PARTITION rely on constraint semantics. They do not match a specific function name, so this would be a new mechanism, and the material does not yet justify it.

Thread: [#32 (Matthias van de Meent, 2026-09-22)](https://www.postgresql.org/message-id/CAEze2WjOOzLVxV%2Bf%3D7DVcky2FqoxchAVikhTeRmYJhnv0KnOcw%40mail.gmail.com), [#39 (Laurenz Albe, 2026-10-02)](https://www.postgresql.org/message-id/e569761f7ba04e3dd2ae947e4ceca5b440009161.camel%40cybertec.at), [#40 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2Wirgxfh7puw_9WTGyg6Aps1HPTHMAQ2TEJUjO0Tej4ZYQ%40mail.gmail.com), [#42 (Alberto Piai, 2026-11-06)](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com)

**Question:** Is the pg_datum_image_equal shape a contract Postgres can keep across releases, and what is the accepted form if that function is not committed as proposed?

Counter-check: held. The runbook scenario is weak because no release has shipped the documented form. The dependency is real: v12 cannot be committed before v12-0001, and message #42 leaves the fallback constraint shape open.

### 3. Constraint left behind after conversion conflicts with later SET EXPRESSION

fixable · level 2, rubric item 13 · design, pg_dump / restore round-trip, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An operator converts column b using CHECK (pg_datum_image_equal(b, a * 2)) and keeps the constraint, as v12 now does. Months later someone runs SET EXPRESSION AS (a * 3). The table rewrite checks the old constraint, which fails on every row, and the error names a constraint the operator no longer remembers.

Following message #8 and message #12, v12 no longer drops the constraint, citing SET NOT NULL and ATTACH PARTITION. In those cases the leftover constraint stays true after any later change to the column. Here it duplicates the generation expression, so the two can drift apart through SET EXPRESSION or DROP EXPRESSION. The documentation hunk does not say what to do with the constraint afterwards, and it does not mention this interaction.

Thread: [#8 (Laurenz Albe, 2026-06-15)](https://www.postgresql.org/message-id/3af338791e8ab27c4a7efbc4217c97162de76bc8.camel%40cybertec.at), [#10 (Alberto Piai, 2026-06-30)](https://www.postgresql.org/message-id/DJMFASI8OYBS.2JBMQ7KZ2XOTO%40gmail.com), [#12 (Laurenz Albe, 2026-07-07)](https://www.postgresql.org/message-id/2eb68e74ea95fc2f691de7d8cd28448d00d1d216.camel%40cybertec.at), [#42 (Alberto Piai, 2026-11-06)](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com)

**Question:** What happens to the leftover constraint under SET EXPRESSION, DROP EXPRESSION and pg_dump, and why should it stay instead of being dropped or documented as removable?

Counter-check: held. v12 keeps the constraint and the documentation hunk does not say what to do with it afterwards. A later SET EXPRESSION with a different expression would fail the rewrite on that constraint, and the brief lists this interaction as open.

### 4. Subscribers silently stop receiving the converted column

fixable · level 2, rubric item 13 · WAL / replication / standby · `replication-ondisk-2`

A publication uses the default, which does not publish generated columns. The column reaches the subscriber during the backfill. After the conversion runs on the publisher, new and updated rows arrive without it, and the subscriber column keeps NULL or its default. Nobody sees an error, so the drift shows up later in reports or after a failover to the subscriber.

Message #10 describes this hazard and a workaround with triggers on the subscriber, but the TAP tests were left out. The alter_table.sgml hunk in v12 does not mention logical replication. With ADD COLUMN the column is new on both sides. Here the column changes from replicated to not replicated in the middle of a migration that operators run online.

Thread: [#10 (Alberto Piai, 2026-06-30)](https://www.postgresql.org/message-id/DJMFASI8OYBS.2JBMQ7KZ2XOTO%40gmail.com)

**Question:** Will the documentation and tests cover the procedure for subscribers when generated columns are not published, or will the command warn when the table is in such a publication?

Counter-check: held. Message #10 describes the hazard and a workaround with subscriber triggers, but the v12 documentation hunk does not mention replication. An operator who follows the documented recipe would not learn of it.

### 5. Restriction list must mirror every generated-column rule by hand

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-1`

A later release adds a new rule that CREATE TABLE and ADD COLUMN enforce for stored generated columns. The new conversion path is not updated, so a user converts a column into a state that CREATE TABLE would reject. They find out when pg_dump output fails to restore.

The command reaches a catalog state through a second path, guarded by its own list of exclusions: identity, serial, partition key, FK actions, foreign tables, references to other generated columns. Message #42 adds two of these (FK actions, atthasmissing) after the patch had been marked Ready for Committer, which shows the list drifts. The proposal does not say whether these checks are shared with the CREATE TABLE code or copied, or what fails if a future rule is missed.

Thread: [#22 (Laurenz Albe, 2026-08-28)](https://www.postgresql.org/message-id/31fec16020bdf25d4a62cbbb4b8ea00af007e0d9.camel%40cybertec.at), [#42 (Alberto Piai, 2026-11-06)](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com)

**Question:** Are the eligibility checks shared with the CREATE TABLE and ADD COLUMN code, so that a new generated-column restriction applies here without anyone changing this command?

Counter-check: held. Message #42 added the FK-action and foreign-table restrictions after the patch was marked Ready for Committer, so the list has already missed rules. The material does not say whether the checks are shared with the CREATE TABLE path.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The command trusts existing heap values without a scan and clears attmissingval without a rewrite. |
| WAL / replication / standby | **yes** | The converted column stops being sent to subscribers when the publication does not publish generated columns. |
| wire protocol | no |  |
| catalog changes | **yes** | v12-0001 adds a pg_proc entry, and the command changes pg_attribute and pg_attrdef for the column. |
| ABI / extension API | **yes** | The patch changes parsenodes.h for the new ALTER TABLE subcommand, which extensions that inspect utility statements see. |
| SQL-standard conflict or non-standard syntax | **yes** | New non-standard syntax ADD GENERATED USING CONSTRAINT ... STORED; message #5 raised a possible future conflict with the standard. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Restore recomputes the column from the expression, and the leftover check constraint is dumped with the table. |
| psql UX conventions | **yes** | The patch adds psql tab completion for the new syntax. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second path to the generated-column catalog state carries its own restriction list and a constraint matched by function name. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What guarantees that no tuple still reads its value from attmissingval when the command clears it, or does the command refuse when atthasmissing is set?
2. What constraint shape does the command accept if pg_datum_image_equal is not committed as proposed, and can Postgres keep that shape across releases?
3. What happens to the leftover constraint under a later SET EXPRESSION or DROP EXPRESSION, and why keep it after the conversion?
4. Will the documentation or the command warn when the table is in a publication that does not publish generated columns?
5. Are the eligibility checks shared with the CREATE TABLE and ADD COLUMN code, or copied into this command?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera is a registered reviewer and commented in message #33, but has not replied to v12 or given a view on the syntax. The owner needs to know ALTER TABLE, generated columns and fast defaults.
- **Liability:** The owner signs for the claim that a valid constraint proves the stored values match the expression, with no scan to catch a mistake. The syntax and the required constraint shape are permanent once released.
- **Author follow-up:** The author has posted twelve versions since March 2026 and withdrew the Ready for Committer status after finding a soundness flaw in the patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Operators of large tables need this, and reviewers and conference attendees confirmed the need. The main patch is about 74 KB in tablecmds.c and gram.y and depends on a separate uncommitted function, and the soundness rule changed again in v12.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` Logical replication silently stops carrying the column's values. Refuted (not specific): Duplicate of replication-ondisk-2, which states the subscriber outcome more exactly and also notes that the TAP tests were left out.
- `maintenance-fit-2` Soundness rests on an uncommitted function with no committer view. Refuted (not specific): Duplicate of standards-compat-2 on the dependency and the fallback. The remaining point about committer ownership belongs in the committability field, and a committer, Álvaro Herrera, is a registered reviewer.
- `already-exists-1` Virtual generated columns or trigger-maintained columns may already cover the need. Refuted (already handled): Virtual columns recompute the expression on every read and do not allow user-defined functions or types, so they do not serve cases like the tsvector column in message #37. Laurenz Albe, Álvaro Herrera and conference attendees (messages #5, #6) independently recognized the need.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the thread cites no earlier attempt at this idea. Message #2 points to the DROP EXPRESSION inheritance discussion and message #32 to the separate pg_datum_image_equal patch, but both are related work, not earlier rejections.
- standards-compat: 3 objection(s). The thread raised a possible future conflict between this syntax and the SQL standard (message #5). Laurenz Albe accepted the syntax, so it is not raised here, but no committer has given a view on it yet.
- replication-ondisk: 2 objection(s). The change is catalog-only: pg_attribute, pg_attrdef and the leftover constraint. It adds no WAL record and no wire protocol change. The dump and restore path is covered by the image equality check.
- security: 0 objection(s). The proposal adds no new privilege or exposure. Only the table owner can run the new ALTER TABLE subcommand, and that owner can already add the same stored generated column with a rewrite. The generation expression must be immutable and comes from a CHECK constraint that is already valid on the owner's table. pg_datum_image_equal() compares only values the caller can already read, and NULL handling no longer lets concurrent writers slip rows past the check (message #25). Without the code I cannot check whether pg_datum_image_equal() is marked leakproof, and if it is, whether it can raise errors that depend on the data.
- maintenance-fit: 2 objection(s). There is no hot-path cost for non-users: the command only runs when invoked, and the leftover constraint can be dropped by the user.
- already-exists: 1 objection(s). No contrib module or external tool turns a column into a generated column without a rewrite. The only existing alternatives are the in-core ones named above.
- demand: 0 objection(s). The material shows enough demand: the author hit the problem in production (message #1), several pgconf.dev attendees called it a real operational pain point (message #6), and Laurenz Albe and Álvaro Herrera accepted the use case. Costs fall mostly on users of the new subcommand. The dependency on pg_datum_image_equal and the syntax are design questions for other angles.

### Precedents retrieved

- P1: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P2: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P3: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P4: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P5: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P6: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P7: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P8: [format_datum debugging function](https://commitfest.postgresql.org/patch/5165/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Revert: Allow table AM to store complex data structures in rd_amcache](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=922c4c461d213a422ee7eb6c38e399607539210a) (revert, Reverted, 2024-04-11) - judged not relevant
- P10: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P11: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P12: [revert "warn of SECURITY DEFINER schemas for non-sql_body funcs"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f77becbe5a99c9853b1ae7f22aa15babd01368b7) (revert, Reverted, 2022-09-28) - judged not relevant
- P13: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P14: [Bug in query rewriter - hasModifyingCTE not getting set](https://commitfest.postgresql.org/patch/3134/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P15: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant
- P16: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P17: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P18: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P19: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P20: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P21: [Force the old transactions logs cleanup even if checkpoint is skipped](https://commitfest.postgresql.org/patch/4619/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P22: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P23: [Online checksums verification](https://commitfest.postgresql.org/patch/2369/) (cf, Returned with feedback, 2020-11-05) - judged not relevant
- P24: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 167 s; cost reported by the CLI: $3.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 88 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:59:41.499Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
