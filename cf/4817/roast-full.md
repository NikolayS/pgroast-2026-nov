# Design pre-review: COPY (on_error table) saving error info to a table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4817](https://commitfest.postgresql.org/patch/4817/) (Needs review); authors: Jian He (stayreal)
- Thread: [on_error table, saving error info to a table](https://www.postgresql.org/message-id/flat/CACJufxH_OJpVra%3D0c4ow8fbxHj7heMcVaTNEPa5vAurSeNA-6Q%40mail.gmail.com), started 2024-02-03
- Messages read: 31 of 31
- Patch prose read: v15-0002-COPY-FROM-on_error-table-error_table-errtbl.patch, v15-0001-export-ExecInsert.patch, v1-0001-on_error-table-saving-error-info-to-a-table.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal builds on the existing ON_ERROR soft-error path and no surviving objection is fatal. The design has open points that the list must settle before code review: replication-ondisk-1 (no rule for changing the fixed pg_catalog type under existing typed tables) and maintenance-fit-2 (COPY builds ModifyTable state by hand around an exported ExecInsert). standards-compat-2 and standards-compat-3 are smaller fixable points. Confidence is medium because only 3 of 31 messages were visible.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the hand-built executor state, the preliminary lock check and the single-row insert mode in v15 could not be checked. Messages #4 to #31 were seen only through the objections' references.

## The proposal, restated

**Problem.** With on_error ignore, COPY FROM skips malformed rows but keeps only NOTICE output, so users cannot query which lines and fields failed. The author wants each failed row's error details stored in a table. The thread builds on an earlier suggestion by Tom Lane, linked but not quoted.

**Proposed design.** COPY FROM gets a new on_error value, table, plus an ERROR_TABLE option that names a user-created typed table of the new built-in composite type pg_catalog.copy_error_saving. On each input conversion error, COPY builds a row of error metadata and inserts it through an exported ExecInsert, so constraints and triggers apply. Then COPY moves on to the next row.

**User-visible surface.**
- COPY option value ON_ERROR table, for COPY FROM in text or csv format only.
- New COPY option ERROR_TABLE error_saving_table, required with ON_ERROR table and allowed only with it.
- Built-in composite type pg_catalog.copy_error_saving with 11 columns (starttime, userid, copy_tbl, filename, lineno, line, colname, raw_field_value, err_message, err_detail, errorcode).
- New NOTICE messages giving the count of saved rows and, with LOG_VERBOSITY verbose, one message per saved row.
- C API: ExecInsert in nodeModifyTable.c becomes exported.

**What the author claims.**
- A preliminary lock check at COPY start means error row inserts cannot block or deadlock in the middle of the COPY.
- Using a typed table makes table validation simpler, since only reloftype needs to be checked.
- Routing inserts through ExecInsert is neater and close to the normal insert path.
- The copy_error_saving definition is fixed, and ALTER TYPE on it is rejected.
- The starttime column tells apart rows saved by different COPY commands into one table.

**Thread state.** Nishant Sharma, Kirill Reshke, Andrew Dunstan, Solaimurugan V and Zsolt Parragi reviewed; Zsolt said v13 looked generally good, and the author says the design may not be bulletproof yet. Open at v15: the NOTICE count is wrong when a BEFORE ROW trigger on the error table returns NULL, and the patch needs a rebase.

## 2. Prior attempts

### Handling of the input data errors in COPY FROM (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1391/](https://commitfest.postgresql.org/patch/1391/)
- **Thread:** [https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com](https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com)
- **Date:** decided 2018-03-03; started 2017-03-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Stephen Frost, Craig Ringer, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Catching the error without PG_RE_THROW() skips the cleanup that AbortTransaction() does, such as resetting memory contexts, releasing locks and restoring user and security contexts. Input functions, including ones from extensions such as PostGIS, can do arbitrary work, so this is not safe. Reviewers suggested a subtransaction approach that retries a batch and skips the failed row. Peter Eisentraut returned the patch because it did not do this or a safe 'soft error' scheme, which Tom Lane also judged hard to make safe.
- **Reason (quote):** "In the end, this isn't an acceptable approach to this problem." ([message](https://www.postgresql.org/message-id/20180123022122.GD2416%40tamriel.snowman.net))
- **What is different this time:** P2 caught input-function errors with PG_TRY() and did not re-throw them. This proposal uses the existing ON_ERROR soft-error path and adds an ordinary executor insert of the error metadata.

### Handling of the input data errors in COPY FROM (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1179/](https://commitfest.postgresql.org/patch/1179/)
- **Thread:** [https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com](https://www.postgresql.org/message-id/flat/7179F2FD-49CE-4093-AE14-1B26C5DFB0DA@gmail.com)
- **Date:** decided 2017-11-30; started 2017-03-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Alvaro Herrera (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier returned the entry because the last patch was five months old, no longer applied, and failed "git diff master --check". On the rebased patch, Alvaro Herrera objected that catching errors in a PG_TRY block and only logging a WARNING is unsafe. Out-of-memory, WAL write failures and serialization failures from triggers can leave memory or session state broken until the transaction aborts. Earlier, Peter Geoghegan said a design that uses subtransactions in a way not obvious to the user would be hard to sell to a committer.
- **Reason (quote):** "After looking at the way you're handling errors, it seems that you've chosen to go precisely the way people were saying that was not admissible: use a PG_TRY block, and when things go south, simply log a WARNING and move on. This is unsafe." ([message](https://www.postgresql.org/message-id/20171130225836.jcabecj6izpp4wyt%40alvherre.pgsql))
- **What is different this time:** P1 wrapped InputFunctionCall in PG_TRY/PG_CATCH and logged a WARNING. This proposal reuses the soft-error reporting of ON_ERROR ignore, so the unsafe-catch objection does not carry over.


## 3. Objections a reviewer is likely to raise

### 1. Fixed pg_catalog row type makes later column changes break upgrades

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 6 · on-disk format / pg_upgrade, catalog changes, pg_dump / restore round-trip · `replication-ondisk-1`

A user creates err_tbl as `create table err_tbl of copy_error_saving` and fills it. A later release adds, renames or retypes a column of copy_error_saving. pg_upgrade or a dump and restore then has to recreate err_tbl as a typed table of a type that no longer matches the saved rows, and the upgrade or the data load fails.

The type is created by system_functions.sql, so each release defines it fresh, but user tables bound to it carry the old layout. Message #15 names this compatibility risk and asks for confidence that the definition will not change. Message #30 then added starttime, so the definition changed during review. Message #14 says a built-in type needs broader consensus, and no committer has given it.

Thread: [#14 (jian he, 2025-08-13)](https://www.postgresql.org/message-id/CACJufxEo-rsH5v__S3guUhDdXjakC7m7N5wj%3DmOB5rPiySBoQg%40mail.gmail.com), [#15 (jian he, 2025-10-22)](https://www.postgresql.org/message-id/CACJufxFVNyrzMxoDqLpmPgQPAco7v_KRUP%2BZ8L7JBVYCZ-NZ0Q%40mail.gmail.com), [#30 (jian he, 2026-09-30)](https://www.postgresql.org/message-id/CACJufxFRDRkqMhuQqoTXY8-30epEGHim85CS8dMfhRR%2BXK2sUQ%40mail.gmail.com)

**Question:** What happens to existing typed error tables under pg_upgrade and pg_dump when a later release changes copy_error_saving, and how is that change carried out?

Counter-check: held. Message #15 names this risk, and v15 changed the column list without saying how existing typed tables would follow. A new patch version could solve it, for example with a stated rule for changing the type, so the severity is fixable.

### 2. COPY builds its own ModifyTable executor state around an exported ExecInsert

fixable · level 2, rubric item 9 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch changes what ExecInitModifyTable sets up before ExecInsert runs, for example a new ResultRelInfo field or trigger state. COPY's hand-built ModifyTableContext lacks that setup, and no test fails until a user puts an unusual trigger or index on the error table. Then COPY crashes or misbehaves.

Message #19 shows that COPY copies part of ExecInitModifyTable by hand, arguing case by case that partitions, foreign tables, RLS and EvalPlanQual can be skipped. Messages #24 and #25 then found a use-after-close and a trigger-ordering server crash in this code, which shows how easily the two copies drift. ExecInsert also becomes an exported symbol that extensions will start to call. Later ModifyTable changes would have to keep this second caller in sync with nothing to flag a miss.

Thread: [#19 (jian he, 2026-05-06)](https://www.postgresql.org/message-id/CACJufxH_NbPuA%2BO5YR7xP4xDZ%2BiHkO2VFkddhrhBz%2B4-EUTp7w%40mail.gmail.com), [#21 (jian he, 2026-05-11)](https://www.postgresql.org/message-id/CACJufxGBmz%3DW63R0ufSCEauokapWJg%3DYrvvcyjD%2B8ia1N5B-DQ%40mail.gmail.com), [#24 (Zsolt Parragi, 2026-05-14)](https://www.postgresql.org/message-id/CAN4CZFPOj5LhzwrmcvKd8D0v_D6EiKSBFazDBCzzq8zAP5a9Vw%40mail.gmail.com), [#25 (jian he, 2026-05-15)](https://www.postgresql.org/message-id/CACJufxG5M6%2Bb6dkVPVdN%2B%2BiDvVE%3D5iEiymMgJrx711siwa3Rxg%40mail.gmail.com)

**Question:** Can the error-row insert reuse an existing, already supported entry point, so that later ModifyTable changes cannot silently break COPY?

Counter-check: held. Message #19 describes setting up ModifyTable state by hand in COPY, and messages #24 and #25 show a crash and a stale relation in that code. The claim that extensions will call ExecInsert is a guess and should not be used as support.

### 3. copy_tbl and userid stored as bare OIDs do not survive dump/restore

fixable · level 2, rubric item 12 · pg_dump / restore round-trip, on-disk format / pg_upgrade · `standards-compat-2`

An operator dumps a database whose error table holds a week of rejected rows and restores it on a new server. Tables and roles get new OIDs, but copy_tbl and userid keep the old numbers. Queries that join to pg_class or pg_authid then return nothing or name the wrong table, and nothing reports this.

The documentation states there is no dependency on pg_class or pg_authid, and expects only stale values after a DROP. Postgres already has a way to store such references in user tables so they dump as names, for example regclass and regrole. The proposal does not say why plain oid was chosen over these types. Since the error table exists to be kept and queried later, its contents need to round-trip.

Thread: [#1 (jian he, 2024-02-03)](https://www.postgresql.org/message-id/CACJufxH_OJpVra%3D0c4ow8fbxHj7heMcVaTNEPa5vAurSeNA-6Q%40mail.gmail.com), [#15 (jian he, 2025-10-22)](https://www.postgresql.org/message-id/CACJufxFVNyrzMxoDqLpmPgQPAco7v_KRUP%2BZ8L7JBVYCZ-NZ0Q%40mail.gmail.com), [#30 (jian he, 2026-09-30)](https://www.postgresql.org/message-id/CACJufxFRDRkqMhuQqoTXY8-30epEGHim85CS8dMfhRR%2BXK2sUQ%40mail.gmail.com)

**Question:** Why does copy_error_saving use plain oid for copy_tbl and userid, and not regclass and regrole, given that error tables are kept across dump and restore?

Counter-check: held. The datatype.sgml hunk says the OIDs go stale only after a drop and does not cover dump and restore, which assigns new OIDs. regclass and regrole dump as names, and the thread does not say why plain oid was chosen.

### 4. ON_ERROR table cannot be combined with REJECT_LIMIT

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-3`

A load job runs with `on_error ignore, reject_limit 100` and switches to `on_error table` to keep the rejected rows. COPY then rejects REJECT_LIMIT with a FEATURE_NOT_SUPPORTED error, so the job must choose between saving errors and limiting them. A badly broken file of millions of lines loads to the end and fills the error table.

Message #26 shows the patch refuses REJECT_LIMIT with ON_ERROR table. Under the current option set, `table` behaves as a variant of `ignore` that also records each row, so the existing limit applies to it in the same way. The proposal does not say why the limit cannot apply here. Having ERROR_TABLE also adds a second required option where one option implying the other would be enough, and the proposal does not justify this either.

Thread: [#26 (Zsolt Parragi, 2026-05-15)](https://www.postgresql.org/message-id/CAN4CZFPoohFvQTSE0wC%2BwcrfYiZOxFmUdOq0%2B9TCVR6Hk8n6iw%40mail.gmail.com)

**Question:** Why should ON_ERROR table refuse REJECT_LIMIT, when the limit applies to ON_ERROR ignore and both options skip the same rows?

Counter-check: held. Message #26 quotes the check that rejects REJECT_LIMIT with ON_ERROR table, and the thread gives no reason for it. The remark about ERROR_TABLE being a second required option is weaker and should not carry the objection.

### 5. Staging table plus pg_input_error_info already gives the same error rows

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user who wants queryable load errors can COPY into an all-text staging table, then use pg_input_is_valid() and pg_input_error_info() per column to split good rows from bad ones. That puts message, detail and SQLSTATE into an ordinary table they define. If the feature goes in, core keeps a fixed pg_catalog type and an exported ExecInsert for a result users can already build in SQL.

Postgres already ships pg_input_is_valid() and pg_input_error_info(), which use the same soft-error path as ON_ERROR ignore and return the fields the copy_error_saving type stores. The thread does not compare the proposal with this idiom. Message #2 asked why a table is needed at all, and the answer in messages #3 and #5 is about creating the table, not about need. The staging idiom does cost a second write and loses exact line numbers for multi-line CSV fields, so a committer would likely ask whether that gap justifies a permanent catalog type.

Thread: [#2 (Nishant Sharma, 2024-07-15)](https://www.postgresql.org/message-id/CADrsxdYb3HZKBnsAUV%3DH2F_54MUPoqW-VTBbOqFTiNrb%2BLDN7g%40mail.gmail.com), [#3 (jian he, 2024-08-20)](https://www.postgresql.org/message-id/CACJufxGMOVr53-%3DoSMvogbX0BN5R73AD4sj0r1M3P6-x5XNwjg%40mail.gmail.com), [#5 (jian he, 2024-12-03)](https://www.postgresql.org/message-id/CACJufxGcX2fKxQqjXJdnXLSVSyowrGiQ5i0960UbiKytu0gm5A%40mail.gmail.com)

**Question:** Which real loads cannot use a text staging table with pg_input_error_info(), because of size, line numbers or something else, and how common are they?

Counter-check: held. A text staging table with pg_input_error_info() can produce the same error fields, and the thread never compares the two. The material shows no independent user reports that the double write or lost line numbers are a common problem.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | User typed tables take their row layout from the built-in type copy_error_saving, which a later release may change. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | system_functions.sql creates the new composite type pg_catalog.copy_error_saving, and ALTER TYPE on it is rejected. |
| ABI / extension API | **yes** | Patch 0001 exports ExecInsert from nodeModifyTable.c. |
| SQL-standard conflict or non-standard syntax | **yes** | gram.y gains the COPY option ERROR_TABLE and the ON_ERROR value table. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Error tables are typed tables of a system type and store copy_tbl and userid as plain OIDs. |
| psql UX conventions | **yes** | The patch changes tab-complete.in.c for the new COPY options. |
| security / privileges | **yes** | COPY checks INSERT privilege on the error table and fires its triggers as the COPY user. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A fixed catalog type and a second caller of ExecInsert with hand-built executor state must be kept working. |
| performance on the hot path | **yes** | The review notes say v15 switches COPY to single-row inserts in some cases, and the thread gives no numbers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A text staging table with pg_input_error_info() gives similar error rows today. |

## 5. Questions the author must answer before anyone reviews code

1. How would a later release add or change a copy_error_saving column without breaking pg_upgrade or dump and restore of existing typed error tables?
2. Can the error-row insert use an existing supported entry point, so that later ModifyTable changes cannot silently break COPY's hand-built executor state?
3. Does a failed insert into the error table (constraint violation, trigger error) abort the whole COPY, and is that the intended behaviour?
4. Which real loads cannot use a text staging table with pg_input_error_info(), because of size, line numbers or something else?
5. Why does copy_error_saving store copy_tbl and userid as plain oid and not as regclass and regrole?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows both COPY FROM and the ModifyTable executor code. The one committer post, message #8, suggests typed tables and does not review the patch as a whole.
- **Liability:** The owner signs for a pg_catalog type whose columns are frozen for existing user tables across all later releases. The owner also signs for keeping COPY's hand-built insert path in step with ModifyTable changes.
- **Author follow-up:** The author posted 15 versions since 2024 and keeps moving the entry between CommitFests, which suggests continued follow-up. The author also says the design may not be bulletproof yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The thread shows reviewer interest but no user reports, and the 65 KiB patch touches COPY, the grammar and the executor. Two design points, the type's upgrade rule and the insert path, need an answer and a committer's view before more line review helps.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-3` (fixable, level 2) No committer who owns COPY or the executor has engaged. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-1` Fixed pg_catalog composite type has no upgrade path for its columns. Refuted (not specific): Duplicate of replication-ondisk-1. The claim that dump and restore "silently realigns the columns" is also doubtful, because pg_dump writes a column list in its COPY commands.
- `replication-ondisk-2` Saved OIDs of the role and target table do not survive dump and restore. Refuted (not specific): Duplicate of standards-compat-2, which names the existing alternative (regclass, regrole).
- `security-1` Error table triggers run with the privileges of the COPY user. Refuted (no failure scenario): The scenario is the same as a superuser running INSERT or COPY into a table owned by a less privileged role, where Postgres already fires triggers as the calling user. The COPY user names the error table in ERROR_TABLE, so the patch follows existing practice.
- `maintenance-fit-1` Fixed pg_catalog composite type ties every future release to its column list. Refuted (not specific): Duplicate of replication-ondisk-1.
- `demand-1` No user demand shown beyond the existing NOTICE and staging-table workarounds. Refuted (not specific): Duplicate of already-exists-1, which asks for the same demand evidence and names the stronger alternative. The verbose NOTICE output is not queryable, so it is not a real substitute.
- `demand-2` A fixed pg_catalog type is a lasting cost for a narrow gain. Refuted (not specific): Duplicate of replication-ondisk-1. The claim that every user of an upgraded cluster bears a cost is not supported, because an unused catalog type costs non-users nothing shown in the material.

### Review angles

- prior-attempts: 0 objection(s). P1 and P2 died because they caught errors with PG_TRY, and this proposal sits on the soft-error path ON_ERROR ignore already uses. P8 logged all COPY data to the server log, while this proposal stores only failed rows in a table the user chose. Message #1 cites an earlier Tom Lane suggestion by link only, so the material does not show its content.
- standards-compat: 3 objection(s). Not applicable: three objections raised.
- replication-ondisk: 2 objection(s). WAL, standby, wire protocol and the ExecInsert export add no new cross-node state, so they raise nothing from this angle.
- security: 1 objection(s). One objection, on trigger execution context. Leaks of raw input lines go only to a table the COPY user chose, so they are not raised. Unbounded error-table growth comes from the user's own input, so it is not raised as a denial of service.
- maintenance-fit: 3 objection(s). Hot-path cost seems limited to users of ON_ERROR table, but the proposal gives no numbers, and v15's switch to CIM_SINGLE in some cases is not explained.
- already-exists: 1 objection(s). One objection only. Other workarounds, such as capturing LOG_VERBOSITY verbose NOTICEs on the client or using external loaders, are weaker substitutes, so they are not raised separately.
- demand: 2 objection(s). Two objections raised. The demand objection is a not-now case: it asks for field evidence and does not argue that the feature is a bad idea.

### Precedents retrieved

- P1: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1179/) (cf, Returned with feedback, 2017-11-30) - used in the review; reason summarised from its thread
- P2: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1391/) (cf, Returned with feedback, 2018-03-03) - used in the review; reason summarised from its thread
- P3: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P4: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P5: [revert "Throw error for ALTER TABLE RESET of an invalid option"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb74a67e26523eb2408f441bfc589c80f76c465) (revert, Reverted, 2014-08-26) - judged not relevant
- P6: [Revert my bad decision of about a year ago to make PortalDefineQuery responsible for copying the query string into the new Portal.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1591fcbec77f6544a0f665758bae912c68e6fa42) (revert, Reverted, 2008-04-02) - judged not relevant
- P7: [Revert an ill-considered portion of my patch of 12-Mar, which tried to save a few lines in sql_exec_error_callback() by using the function source string...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6775c0108039a13458004d893e4a17b4ae3fae9d) (revert, Reverted, 2007-06-17) - judged not relevant
- P8: [File content logging during execution of COPY queries](https://commitfest.postgresql.org/patch/828/) (cf, Rejected, 2016-12-02) - used in the review; reason summarised from its thread
- P9: [Make ON_ERROR_STOP stop on shell script failure](https://commitfest.postgresql.org/patch/3936/) (cf, Returned with feedback, 2023-07-06) - judged not relevant; reason summarised from its thread
- P10: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P11: [psql: default to ON_ERROR_STOP](https://commitfest.postgresql.org/patch/3446/) (cf, Returned with feedback, 2022-01-17) - judged not relevant; reason summarised from its thread
- P12: [Inconsistent error handling in START_REPLICATION command](https://commitfest.postgresql.org/patch/474/) (cf, Rejected, 2016-03-11) - judged not relevant; reason summarised from its thread
- P13: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P15: [Revert "Add a test for transition table usage in FOR EACH ROW trigger."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a19ea9c6601bfb06dfd9f4c1060550dbc3f7bde1) (revert, Reverted, 2017-05-16) - judged not relevant
- P16: [Revert "Add soft error handling to some expression nodes"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8ec5e0543b90372c8e6d5cc2cd3d2ff89ca0e82) (revert, Reverted, 2023-10-02) - judged not relevant
- P17: [Remove useless savepointLevel in TransactionStateData](https://commitfest.postgresql.org/patch/3958/) (cf, Withdrawn, 2022-10-24) - judged not relevant
- P18: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P19: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/2270/) (cf, Returned with feedback, 2020-03-16) - judged not relevant
- P20: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P21: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant
- P22: [Test for function error in logrep worker](https://commitfest.postgresql.org/patch/4046/) (cf, Withdrawn, 2023-04-05) - judged not relevant
- P23: [Simplify search of end of argv in save_ps_display_args()](https://commitfest.postgresql.org/patch/575/) (cf, Rejected, 2016-03-10) - judged not relevant
- P24: [Add Change Badges to documentation](https://commitfest.postgresql.org/patch/2309/) (cf, Returned with feedback, 2019-11-25) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 219 s; cost reported by the CLI: $4.16
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 113 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:28:29.549Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
