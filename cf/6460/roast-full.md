# Design pre-review: Show expression of virtual columns in error messages

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6460](https://commitfest.postgresql.org/patch/6460/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [Show expression of virtual columns in error messages](https://www.postgresql.org/message-id/flat/DG5DV8SED62G.2WFJB46D7WIT8%40gmail.com), started 2026-02-03
- Messages read: 18 of 18
- Patch prose read: v8-0001-Compute-virtual-generated-columns-during-INSERT-U.patch, v1-0001-Show-expression-of-virtual-columns-in-error-messa.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal of a useful DETAIL line is agreed, but v8 also makes write-time computation mandatory for every table with a virtual column. already-exists-1 survived: message #9 raised computing only when constraints exist, and the thread does not say why v8 rejects that. The author lists the consequences in message #17 and asks whether to continue, and nobody has replied. The list has to settle that scope before the level 2 points can be judged.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check that DETAIL hides a virtual column value from a user without SELECT on that column, or that every write path goes through ExecConstraints(). The level 2 objections on restore, apply and overhead were not evaluated in the verdict.

## The proposal, restated

**Problem.** For a virtual generated column, the "Failing row contains" detail of check and not-null violations shows the string "virtual", which does not help users debug. Also, a row can be stored even when its virtual column cannot be computed, for example on integer overflow, and the error only appears when the row is read.

**Proposed design.** v8 changes ExecConstraints() to compute all virtual generated columns of the new row into a separate slot and to check constraints against that slot, even when the table has no constraints. The input slot is not changed, so virtual columns are still stored as nulls. Each generation expression runs once per row, not once per check constraint that uses it.

**User-visible surface.**
- DETAIL of check and not-null violations shows the computed value of a virtual column instead of "virtual".
- INSERT, UPDATE, ON CONFLICT, MERGE, COPY FROM and logical replication apply reject a row whose virtual column values cannot be computed.
- An error in a generation expression now comes before not-null, check, unique and "no partition found" errors.
- Restore of a dump, or initial table sync, fails for a table that has rows whose virtual column cannot be computed.
- Docs for ddl.sgml and CREATE TABLE now say virtual columns are also computed on insert and update.

**What the author claims.**
- Each generation expression is evaluated once per row instead of once per check constraint.
- Every write path is covered because all callers go through ExecConstraints(); foreign tables are not affected.
- Computed values never reach disk because the input slot is not modified.
- The author says the write-path consequences risk breaking existing workflows and asks if the feature is still worth pursuing.

**Thread state.** Tom Lane rejected the v1/v2 approach of showing the expression and proposed computing values on write, which v4 to v8 implement; Peter Eisentraut was committer for a short time and then removed himself. After an offlist review found a COPY regression in v7, the author posted v8 with four new behaviour changes and asked whether to continue, and nobody has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Unconditional write-time computation duplicates what stored columns and checks offer

fixable · level 1b, rubric item 4 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), pg_dump / restore round-trip, WAL / replication / standby · `already-exists-1`

A user runs ALTER COLUMN ... SET EXPRESSION, which leaves some existing rows that cannot be computed. A later dump restore or initial table sync fails on COPY and the table ends up empty, and the user learns this only during the restore.

Users who want write-time computability can already get it two ways: declare the column STORED, or add a CHECK constraint that refers to the virtual column, since check expressions are expanded and evaluated on write. v8 makes this computation mandatory for every table with a virtual column, and message #17 lists the cost: restore and table-sync failures, apply-worker errors, and a new order of errors. Message #11 argues that expensive expressions belong in STORED columns. That argument also means the users who want write-time checks already have STORED, so making the check mandatory for everyone adds little.

Thread: [#9 (Tom Lane, 2026-02-26)](https://www.postgresql.org/message-id/719803.1772131627%40sss.pgh.pa.us), [#11 (Tom Lane, 2026-02-26)](https://www.postgresql.org/message-id/742565.1772144083%40sss.pgh.pa.us), [#17 ("Matheus Alcantara", 2026-10-19)](https://www.postgresql.org/message-id/DM93NEP224GW.5527NY59BBN2%40gmail.com)

**Question:** Why not compute virtual values only when the table has constraints to check, which message #9 suggests, and leave write-time validation to STORED or an explicit CHECK?

Counter-check: held. Message #11 argues for computing without constraints, but it predates the consequences listed in message #17, and nobody has weighed them. Message #9 itself raised computing only when constraints exist, and the thread does not say why v8 rejects that option.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | No format change, but rows that cannot be computed stay on disk after pg_upgrade and the new write path refuses to copy them. |
| WAL / replication / standby | **yes** | Logical replication apply and initial table sync now fail on a row whose virtual column cannot be computed. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes src/include/nodes/execnodes.h, which extensions compile against. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Restore of a dump fails on COPY for a table that holds rows whose virtual column cannot be computed. |
| psql UX conventions | no |  |
| security / privileges | **yes** | DETAIL now shows computed virtual column values, so column privilege rules for the failing-row text apply to them. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A write-time computability rule that ALTER TABLE ... SET EXPRESSION and ADD COLUMN do not enforce must be defended for later DDL paths. |
| performance on the hot path | **yes** | ExecConstraints() evaluates every virtual column on every insert, update and COPY FROM row, even without constraints. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A STORED column or a check constraint on the virtual column already gives write-time validation to users who want it. |

## 5. Questions the author must answer before anyone reviews code

1. Why does v8 compute virtual columns for tables without constraints, when message #9 suggested computing them only when there are constraints to check?
2. How does a user find and fix existing rows that cannot be computed before a dump restore or initial table sync, and should SET EXPRESSION and ADD COLUMN check existing rows?
3. Is a stopped apply worker the intended result when a subscriber-only virtual column without constraints cannot be computed for a replicated row?
4. What is the measured overhead of v8 on COPY FROM and UPDATE for a table with several virtual columns and no constraints?
5. Does the SQL standard require a virtual generated column to be computable when the row is written?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the executor constraint path and logical replication apply would have to own it. Tom Lane proposed the direction in message #11 and Peter Eisentraut removed himself as committer; neither has commented on v8.
- **Liability:** The owner signs for restore failures, stopped apply workers and table sync retries on existing data, and for keeping the write-time rule consistent with later ALTER TABLE paths, with no end date.
- **Author follow-up:** The author posted eight versions over several months and fixed a COPY regression found offlist, but asks in message #17 whether the work is still worth pursuing.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The DETAIL improvement helps users of virtual columns with constraints, and the patch is small and limited to execMain.c and execnodes.h. The unconditional write-time computation adds restore and apply failures that nobody has accepted, so the scope needs a decision first.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Dump restore and table sync fail on rows that master accepts. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Subscriber-only virtual columns can now stop logical replication apply. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Unmeasured per-row cost for all writes to tables with virtual columns. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-3` Generation expression errors now take precedence over constraint errors. Refuted (no failure scenario): The row in the scenario is rejected on master too, and Postgres does not promise which of two errors a row raises first. The objection itself says stored generated columns already behave this way, so v8 copies existing practice.
- `replication-ondisk-1` Rows already on disk can make dump restore and table sync fail. Refuted (not specific): Duplicate of standards-compat-1.
- `replication-ondisk-2` Subscriber-only virtual column can stop logical replication apply. Refuted (not specific): Duplicate of standards-compat-2. Its recovery question also has an answer today: the worker logs the error, and ALTER SUBSCRIPTION ... SKIP or disable_on_error apply as for any constraint failure on apply.
- `maintenance-fit-1` Write-time computability is a half-enforced invariant that breaks restore. Refuted (not specific): Duplicate of standards-compat-1, which also asks whether SET EXPRESSION and ADD COLUMN should check existing rows. The fatal severity does not hold, because a new version could scan on ALTER or compute only when constraints exist.
- `maintenance-fit-3` No committer owns v8's behaviour changes, and the author doubts them. Refuted (no failure scenario): The scenario assumes a commit without a committer who accepted the changes, which cannot happen. The rubric treats a missing owner as a committability note that must not count against the design.
- `demand-1` No user demand shown for computing virtual columns on write. Refuted (already handled): Neither STORED nor a check constraint fixes the "virtual" placeholder, and every participant agrees it is unhelpful. A committer raised the unreadable row himself in message #11 as a mismatch with stored columns; already-exists-1 keeps the workaround argument.
- `demand-2` Existing virtual-column users pay restore and sync failures for a small gain. Refuted (not specific): Duplicate of standards-compat-1 for the restore and sync failures. Its claim that every virtual-column user pays is too broad, because users without uncomputable rows pay only the evaluation cost that maintenance-fit-2 covers.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] is about generated columns in COPY FROM ... WHERE, and its recorded reason does not apply here: v8 computes values inside ExecConstraints(), after BEFORE triggers, and does not filter input rows. The only earlier attempt is this thread's own v1/v2 (show the expression). Tom Lane objected to it in message #9, and v4 to v8 follow his design from message #11.
- standards-compat: 3 objection(s). There are three objections, so no note is needed.
- replication-ondisk: 2 objection(s). v8 adds no WAL record, no catalog change and no on-disk format change, and virtual columns are still stored as nulls. The objections concern rows that already exist and cross-node apply.
- security: 0 objection(s). No security objection found. Generation expressions now also run on write, as the writer, during restore and during apply, but Postgres limits virtual column expressions to built-in functions, so table owners cannot plant code there. Without the code I could not check one point: DETAIL must still hide a virtual column value from a user without SELECT on that column.
- maintenance-fit: 3 objection(s). Objections found. The only precedent concerns COPY WHERE filtering on input data, so it does not apply here.
- already-exists: 1 objection(s). The error-message part has no existing equivalent for virtual columns, so this angle raises nothing against it.
- demand: 2 objection(s). Two objections, both 'not now, show demand' in kind.

### Precedents retrieved

- P1: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P2: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P3: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P4: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P5: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P6: [doc: virtual envs with Pl/Python](https://commitfest.postgresql.org/patch/5308/) (cf, Withdrawn, 2025-02-10) - judged not relevant; reason summarised from its thread
- P7: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P8: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - judged not relevant
- P10: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P11: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P12: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P13: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P14: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P15: [Add a reset_computed_values function in pg_stat_statements](https://commitfest.postgresql.org/patch/2275/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P16: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant
- P17: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P18: [remove deprecated v8.2 containment operators](https://commitfest.postgresql.org/patch/2798/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P19: [\[PATCH\] Add support function for containment operators](https://commitfest.postgresql.org/patch/4304/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P20: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P21: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant
- P22: [Reducing Memory Consumption (aset and generation)](https://commitfest.postgresql.org/patch/3673/) (cf, Rejected, 2022-07-12) - judged not relevant
- P23: [Contains and is contained by operators of inet datatypes](https://commitfest.postgresql.org/patch/805/) (cf, Rejected, 2016-11-30) - judged not relevant
- P24: [Generate Doxygen documentation](https://commitfest.postgresql.org/patch/4641/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 153 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 105 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:52:17.289Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
