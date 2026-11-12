# Design pre-review: COPY (on_error table) saving error info to a table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4817](https://commitfest.postgresql.org/patch/4817/); [thread](https://www.postgresql.org/message-id/flat/CACJufxH_OJpVra%3D0c4ow8fbxHj7heMcVaTNEPa5vAurSeNA-6Q%40mail.gmail.com), 31 of 31 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal builds on the existing ON_ERROR soft-error path and no surviving objection is fatal. The design has open points that the list must settle before code review: replication-ondisk-1 (no rule for changing the fixed pg_catalog type under existing typed tables) and maintenance-fit-2 (COPY builds ModifyTable state by hand around an exported ExecInsert). standards-compat-2 and standards-compat-3 are smaller fixable points. Confidence is medium because only 3 of 31 messages were visible.

Not checked: The code was not read, so the hand-built executor state, the preliminary lock check and the single-row insert mode in v15 could not be checked. Messages #4 to #31 were seen only through the objections' references.

## Prior attempts

- [Handling of the input data errors in COPY FROM](https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com) (similar idea; Returned with feedback, 2018-03-03). "In the end, this isn't an acceptable approach to this problem." ([message](https://www.postgresql.org/message-id/20180123022122.GD2416%40tamriel.snowman.net)) Different now: P2 caught input-function errors with PG_TRY() and did not re-throw them. This proposal uses the existing ON_ERROR soft-error path and adds an ordinary executor insert of the error metadata.
- [Handling of the input data errors in COPY FROM](https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com) (similar idea; Returned with feedback, 2017-11-30). "After looking at the way you're handling errors, it seems that you've chosen to go precisely the way people were saying that was not admissible: use a PG_TRY block, and when things go south, simply log a WARNING and move on. This is unsafe." ([message](https://www.postgresql.org/message-id/20171130225836.jcabecj6izpp4wyt%40alvherre.pgsql)) Different now: P1 wrapped InputFunctionCall in PG_TRY/PG_CATCH and logged a WARNING. This proposal reuses the soft-error reporting of ON_ERROR ignore, so the unsafe-catch objection does not carry over.

## Objections a reviewer is likely to raise

### 1. Fixed pg_catalog row type makes later column changes break upgrades

A user creates err_tbl as `create table err_tbl of copy_error_saving` and fills it. A later release adds, renames or retypes a column of copy_error_saving. pg_upgrade or a dump and restore then has to recreate err_tbl as a typed table of a type that no longer matches the saved rows, and the upgrade or the data load fails.

The type is created by system_functions.sql, so each release defines it fresh, but user tables bound to it carry the old layout. Message #15 names this compatibility risk and asks for confidence that the definition will not change. Message #30 then added starttime, so the definition changed during review. Message #14 says a built-in type needs broader consensus, and no committer has given it.

**Question:** What happens to existing typed error tables under pg_upgrade and pg_dump when a later release changes copy_error_saving, and how is that change carried out?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#14](https://www.postgresql.org/message-id/CACJufxEo-rsH5v__S3guUhDdXjakC7m7N5wj%3DmOB5rPiySBoQg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CACJufxFVNyrzMxoDqLpmPgQPAco7v_KRUP%2BZ8L7JBVYCZ-NZ0Q%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CACJufxFRDRkqMhuQqoTXY8-30epEGHim85CS8dMfhRR%2BXK2sUQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. COPY builds its own ModifyTable executor state around an exported ExecInsert

A later patch changes what ExecInitModifyTable sets up before ExecInsert runs, for example a new ResultRelInfo field or trigger state. COPY's hand-built ModifyTableContext lacks that setup, and no test fails until a user puts an unusual trigger or index on the error table. Then COPY crashes or misbehaves.

Message #19 shows that COPY copies part of ExecInitModifyTable by hand, arguing case by case that partitions, foreign tables, RLS and EvalPlanQual can be skipped. Messages #24 and #25 then found a use-after-close and a trigger-ordering server crash in this code, which shows how easily the two copies drift. ExecInsert also becomes an exported symbol that extensions will start to call. Later ModifyTable changes would have to keep this second caller in sync with nothing to flag a miss.

**Question:** Can the error-row insert reuse an existing, already supported entry point, so that later ModifyTable changes cannot silently break COPY?

<sub>fixable, level 2, ABI / extension API; messages [#19](https://www.postgresql.org/message-id/CACJufxH_NbPuA%2BO5YR7xP4xDZ%2BiHkO2VFkddhrhBz%2B4-EUTp7w%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CACJufxGBmz%3DW63R0ufSCEauokapWJg%3DYrvvcyjD%2B8ia1N5B-DQ%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/CAN4CZFPOj5LhzwrmcvKd8D0v_D6EiKSBFazDBCzzq8zAP5a9Vw%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CACJufxG5M6%2Bb6dkVPVdN%2B%2BiDvVE%3D5iEiymMgJrx711siwa3Rxg%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. copy_tbl and userid stored as bare OIDs do not survive dump/restore

An operator dumps a database whose error table holds a week of rejected rows and restores it on a new server. Tables and roles get new OIDs, but copy_tbl and userid keep the old numbers. Queries that join to pg_class or pg_authid then return nothing or name the wrong table, and nothing reports this.

The documentation states there is no dependency on pg_class or pg_authid, and expects only stale values after a DROP. Postgres already has a way to store such references in user tables so they dump as names, for example regclass and regrole. The proposal does not say why plain oid was chosen over these types. Since the error table exists to be kept and queried later, its contents need to round-trip.

**Question:** Why does copy_error_saving use plain oid for copy_tbl and userid, and not regclass and regrole, given that error tables are kept across dump and restore?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/CACJufxH_OJpVra%3D0c4ow8fbxHj7heMcVaTNEPa5vAurSeNA-6Q%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CACJufxFVNyrzMxoDqLpmPgQPAco7v_KRUP%2BZ8L7JBVYCZ-NZ0Q%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CACJufxFRDRkqMhuQqoTXY8-30epEGHim85CS8dMfhRR%2BXK2sUQ%40mail.gmail.com); `standards-compat-2`</sub>

### 4. ON_ERROR table cannot be combined with REJECT_LIMIT

A load job runs with `on_error ignore, reject_limit 100` and switches to `on_error table` to keep the rejected rows. COPY then rejects REJECT_LIMIT with a FEATURE_NOT_SUPPORTED error, so the job must choose between saving errors and limiting them. A badly broken file of millions of lines loads to the end and fills the error table.

Message #26 shows the patch refuses REJECT_LIMIT with ON_ERROR table. Under the current option set, `table` behaves as a variant of `ignore` that also records each row, so the existing limit applies to it in the same way. The proposal does not say why the limit cannot apply here. Having ERROR_TABLE also adds a second required option where one option implying the other would be enough, and the proposal does not justify this either.

**Question:** Why should ON_ERROR table refuse REJECT_LIMIT, when the limit applies to ON_ERROR ignore and both options skip the same rows?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#26](https://www.postgresql.org/message-id/CAN4CZFPoohFvQTSE0wC%2BwcrfYiZOxFmUdOq0%2B9TCVR6Hk8n6iw%40mail.gmail.com); `standards-compat-3`</sub>

### 5. Staging table plus pg_input_error_info already gives the same error rows

A user who wants queryable load errors can COPY into an all-text staging table, then use pg_input_is_valid() and pg_input_error_info() per column to split good rows from bad ones. That puts message, detail and SQLSTATE into an ordinary table they define. If the feature goes in, core keeps a fixed pg_catalog type and an exported ExecInsert for a result users can already build in SQL.

Postgres already ships pg_input_is_valid() and pg_input_error_info(), which use the same soft-error path as ON_ERROR ignore and return the fields the copy_error_saving type stores. The thread does not compare the proposal with this idiom. Message #2 asked why a table is needed at all, and the answer in messages #3 and #5 is about creating the table, not about need. The staging idiom does cost a second write and loses exact line numbers for multi-line CSV fields, so a committer would likely ask whether that gap justifies a permanent catalog type.

**Question:** Which real loads cannot use a text staging table with pg_input_error_info(), because of size, line numbers or something else, and how common are they?

<sub>fixable, level 1b, desirability; messages [#2](https://www.postgresql.org/message-id/CADrsxdYb3HZKBnsAUV%3DH2F_54MUPoqW-VTBbOqFTiNrb%2BLDN7g%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CACJufxGMOVr53-%3DoSMvogbX0BN5R73AD4sj0r1M3P6-x5XNwjg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CACJufxGcX2fKxQqjXJdnXLSVSyowrGiQ5i0960UbiKytu0gm5A%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. How would a later release add or change a copy_error_saving column without breaking pg_upgrade or dump and restore of existing typed error tables?
2. Can the error-row insert use an existing supported entry point, so that later ModifyTable changes cannot silently break COPY's hand-built executor state?
3. Does a failed insert into the error table (constraint violation, trigger error) abort the whole COPY, and is that the intended behaviour?
4. Which real loads cannot use a text staging table with pg_input_error_info(), because of size, line numbers or something else?
5. Why does copy_error_saving store copy_tbl and userid as plain oid and not as regclass and regrole?

## Touch points

- **on-disk format / pg_upgrade:** User typed tables take their row layout from the built-in type copy_error_saving, which a later release may change.
- **catalog changes:** system_functions.sql creates the new composite type pg_catalog.copy_error_saving, and ALTER TYPE on it is rejected.
- **ABI / extension API:** Patch 0001 exports ExecInsert from nodeModifyTable.c.
- **SQL-standard conflict or non-standard syntax:** gram.y gains the COPY option ERROR_TABLE and the ON_ERROR value table.
- **pg_dump / restore round-trip:** Error tables are typed tables of a system type and store copy_tbl and userid as plain OIDs.
- **psql UX conventions:** The patch changes tab-complete.in.c for the new COPY options.
- **security / privileges:** COPY checks INSERT privilege on the error table and fires its triggers as the COPY user.
- **maintenance burden ("who maintains this in 10 years"):** A fixed catalog type and a second caller of ExecInsert with hand-built executor state must be kept working.
- **performance on the hot path:** The review notes say v15 switches COPY to single-row inserts in some cases, and the thread gives no numbers.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A text staging table with pg_input_error_info() gives similar error rows today.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows both COPY FROM and the ModifyTable executor code. The one committer post, message #8, suggests typed tables and does not review the patch as a whole.
- Priority: fix design first; audience unclear; review cost medium. The thread shows reviewer interest but no user reports, and the 65 KiB patch touches COPY, the grammar and the executor. Two design points, the type's upgrade rule and the insert path, need an answer and a committer's view before more line review helps.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
