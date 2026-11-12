# Design pre-review: Adding a stored generated column without long-lived locks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6674](https://commitfest.postgresql.org/patch/6674/); [thread](https://www.postgresql.org/message-id/flat/abkrpUwlGngF4e-d%40phidippus.sen.work), 42 of 42 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread and no objection to it survived. The design is not settled. replication-ondisk-1 describes a path where clearing attmissingval without a rewrite turns existing values into NULL. standards-compat-2 shows the required constraint shape depends on an uncommitted function, with the fallback left open in message #42. The proof rule has changed three times, so the list should agree on it before code review.

Not checked: Without the code I could not check how v12 handles atthasmissing, which lock the command takes, or whether the eligibility checks are shared with CREATE TABLE. Only 3 of the 42 messages were visible, so later messages are known only through the objections.

## Prior attempts

None found before 2026-03-17.

## Objections a reviewer is likely to raise

### 1. Clearing attmissingval can turn unrewritten tuples to NULL

An operator runs `alter table t add column c int not null default 0`, which uses a fast default with no rewrite. The backfill updates only rows `where c is distinct from expr`, VALIDATE passes, and the conversion clears attmissingval. Old tuples whose expr is 0 were never updated, so they now read NULL in a not-null column.

Message #42 says v12 clears atthasmissing and attmissingval for consistency with CREATE TABLE, and the command does no scan or rewrite. For tuples written before the ADD COLUMN, attmissingval is the only source of the column value. The proposal does not say why every tuple must physically hold the column before the catalog entry is removed. A batched, idempotent backfill that skips unchanged rows is a common pattern.

**Question:** What guarantees that no tuple still depends on attmissingval when it is cleared, or does the command refuse when atthasmissing is set?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#42](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com); `replication-ondisk-1`</sub>

### 2. DDL contract depends on an uncommitted function recognized by shape

Operators write migration runbooks against the documented form CHECK (pg_datum_image_equal(col, expr)). If the separate patch lands under another name, with other NULL semantics, or does not land, the documented form changes. In the ROW()::record *= fallback, every runbook written against the earlier form then fails with "not of the form".

In v12 the user-facing contract of the command is a literal function call inside a CHECK constraint, and that function comes from v12-0001, which has its own review (message #42). In message #42 the fallback is left open. Existing proof shortcuts such as SET NOT NULL and ATTACH PARTITION rely on constraint semantics. They do not match a specific function name, so this would be a new mechanism, and the material does not yet justify it.

**Question:** Is the pg_datum_image_equal shape a contract Postgres can keep across releases, and what is the accepted form if that function is not committed as proposed?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#32](https://www.postgresql.org/message-id/CAEze2WjOOzLVxV%2Bf%3D7DVcky2FqoxchAVikhTeRmYJhnv0KnOcw%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/e569761f7ba04e3dd2ae947e4ceca5b440009161.camel%40cybertec.at), [#40](https://www.postgresql.org/message-id/CAEze2Wirgxfh7puw_9WTGyg6Aps1HPTHMAQ2TEJUjO0Tej4ZYQ%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com); `standards-compat-2`</sub>

### 3. Constraint left behind after conversion conflicts with later SET EXPRESSION

An operator converts column b using CHECK (pg_datum_image_equal(b, a * 2)) and keeps the constraint, as v12 now does. Months later someone runs SET EXPRESSION AS (a * 3). The table rewrite checks the old constraint, which fails on every row, and the error names a constraint the operator no longer remembers.

Following message #8 and message #12, v12 no longer drops the constraint, citing SET NOT NULL and ATTACH PARTITION. In those cases the leftover constraint stays true after any later change to the column. Here it duplicates the generation expression, so the two can drift apart through SET EXPRESSION or DROP EXPRESSION. The documentation hunk does not say what to do with the constraint afterwards, and it does not mention this interaction.

**Question:** What happens to the leftover constraint under SET EXPRESSION, DROP EXPRESSION and pg_dump, and why should it stay instead of being dropped or documented as removable?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/3af338791e8ab27c4a7efbc4217c97162de76bc8.camel%40cybertec.at), [#10](https://www.postgresql.org/message-id/DJMFASI8OYBS.2JBMQ7KZ2XOTO%40gmail.com), [#12](https://www.postgresql.org/message-id/2eb68e74ea95fc2f691de7d8cd28448d00d1d216.camel%40cybertec.at), [#42](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com); `standards-compat-1`</sub>

### 4. Subscribers silently stop receiving the converted column

A publication uses the default, which does not publish generated columns. The column reaches the subscriber during the backfill. After the conversion runs on the publisher, new and updated rows arrive without it, and the subscriber column keeps NULL or its default. Nobody sees an error, so the drift shows up later in reports or after a failover to the subscriber.

Message #10 describes this hazard and a workaround with triggers on the subscriber, but the TAP tests were left out. The alter_table.sgml hunk in v12 does not mention logical replication. With ADD COLUMN the column is new on both sides. Here the column changes from replicated to not replicated in the middle of a migration that operators run online.

**Question:** Will the documentation and tests cover the procedure for subscribers when generated columns are not published, or will the command warn when the table is in such a publication?

<sub>fixable, level 2, WAL / replication / standby; messages [#10](https://www.postgresql.org/message-id/DJMFASI8OYBS.2JBMQ7KZ2XOTO%40gmail.com); `replication-ondisk-2`</sub>

### 5. Restriction list must mirror every generated-column rule by hand

A later release adds a new rule that CREATE TABLE and ADD COLUMN enforce for stored generated columns. The new conversion path is not updated, so a user converts a column into a state that CREATE TABLE would reject. They find out when pg_dump output fails to restore.

The command reaches a catalog state through a second path, guarded by its own list of exclusions: identity, serial, partition key, FK actions, foreign tables, references to other generated columns. Message #42 adds two of these (FK actions, atthasmissing) after the patch had been marked Ready for Committer, which shows the list drifts. The proposal does not say whether these checks are shared with the CREATE TABLE code or copied, or what fails if a future rule is missed.

**Question:** Are the eligibility checks shared with the CREATE TABLE and ADD COLUMN code, so that a new generated-column restriction applies here without anyone changing this command?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#22](https://www.postgresql.org/message-id/31fec16020bdf25d4a62cbbb4b8ea00af007e0d9.camel%40cybertec.at), [#42](https://www.postgresql.org/message-id/DMO4P471X6Q0.FTWMWFEUG62W%40gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What guarantees that no tuple still reads its value from attmissingval when the command clears it, or does the command refuse when atthasmissing is set?
2. What constraint shape does the command accept if pg_datum_image_equal is not committed as proposed, and can Postgres keep that shape across releases?
3. What happens to the leftover constraint under a later SET EXPRESSION or DROP EXPRESSION, and why keep it after the conversion?
4. Will the documentation or the command warn when the table is in a publication that does not publish generated columns?
5. Are the eligibility checks shared with the CREATE TABLE and ADD COLUMN code, or copied into this command?

## Touch points

- **on-disk format / pg_upgrade:** The command trusts existing heap values without a scan and clears attmissingval without a rewrite.
- **WAL / replication / standby:** The converted column stops being sent to subscribers when the publication does not publish generated columns.
- **catalog changes:** v12-0001 adds a pg_proc entry, and the command changes pg_attribute and pg_attrdef for the column.
- **ABI / extension API:** The patch changes parsenodes.h for the new ALTER TABLE subcommand, which extensions that inspect utility statements see.
- **SQL-standard conflict or non-standard syntax:** New non-standard syntax ADD GENERATED USING CONSTRAINT ... STORED; message #5 raised a possible future conflict with the standard.
- **pg_dump / restore round-trip:** Restore recomputes the column from the expression, and the leftover check constraint is dumped with the table.
- **psql UX conventions:** The patch adds psql tab completion for the new syntax.
- **maintenance burden ("who maintains this in 10 years"):** A second path to the generated-column catalog state carries its own restriction list and a constraint matched by function name.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera is a registered reviewer and commented in message #33, but has not replied to v12 or given a view on the syntax. The owner needs to know ALTER TABLE, generated columns and fast defaults.
- Priority: fix design first; audience a sizeable minority; review cost medium. Operators of large tables need this, and reviewers and conference attendees confirmed the need. The main patch is about 74 KB in tablecmds.c and gram.y and depends on a separate uncommitted function, and the soundness rule changed again in v12.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
