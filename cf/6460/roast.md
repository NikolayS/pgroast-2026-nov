# Design pre-review: Show expression of virtual columns in error messages

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6460](https://commitfest.postgresql.org/patch/6460/); [thread](https://www.postgresql.org/message-id/flat/DG5DV8SED62G.2WFJB46D7WIT8%40gmail.com), 18 of 18 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal of a useful DETAIL line is agreed, but v8 also makes write-time computation mandatory for every table with a virtual column. already-exists-1 survived: message #9 raised computing only when constraints exist, and the thread does not say why v8 rejects that. The author lists the consequences in message #17 and asks whether to continue, and nobody has replied. The list has to settle that scope before the level 2 points can be judged.

Not checked: Without the code, I could not check that DETAIL hides a virtual column value from a user without SELECT on that column, or that every write path goes through ExecConstraints(). The level 2 objections on restore, apply and overhead were not evaluated in the verdict.

## Prior attempts

None found before 2026-02-03.

## Objections a reviewer is likely to raise

### 1. Unconditional write-time computation duplicates what stored columns and checks offer

A user runs ALTER COLUMN ... SET EXPRESSION, which leaves some existing rows that cannot be computed. A later dump restore or initial table sync fails on COPY and the table ends up empty, and the user learns this only during the restore.

Users who want write-time computability can already get it two ways: declare the column STORED, or add a CHECK constraint that refers to the virtual column, since check expressions are expanded and evaluated on write. v8 makes this computation mandatory for every table with a virtual column, and message #17 lists the cost: restore and table-sync failures, apply-worker errors, and a new order of errors. Message #11 argues that expensive expressions belong in STORED columns. That argument also means the users who want write-time checks already have STORED, so making the check mandatory for everyone adds little.

**Question:** Why not compute virtual values only when the table has constraints to check, which message #9 suggests, and leave write-time validation to STORED or an explicit CHECK?

<sub>fixable, level 1b, desirability; messages [#9](https://www.postgresql.org/message-id/719803.1772131627%40sss.pgh.pa.us), [#11](https://www.postgresql.org/message-id/742565.1772144083%40sss.pgh.pa.us), [#17](https://www.postgresql.org/message-id/DM93NEP224GW.5527NY59BBN2%40gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Why does v8 compute virtual columns for tables without constraints, when message #9 suggested computing them only when there are constraints to check?
2. How does a user find and fix existing rows that cannot be computed before a dump restore or initial table sync, and should SET EXPRESSION and ADD COLUMN check existing rows?
3. Is a stopped apply worker the intended result when a subscriber-only virtual column without constraints cannot be computed for a replicated row?
4. What is the measured overhead of v8 on COPY FROM and UPDATE for a table with several virtual columns and no constraints?
5. Does the SQL standard require a virtual generated column to be computable when the row is written?

## Touch points

- **on-disk format / pg_upgrade:** No format change, but rows that cannot be computed stay on disk after pg_upgrade and the new write path refuses to copy them.
- **WAL / replication / standby:** Logical replication apply and initial table sync now fail on a row whose virtual column cannot be computed.
- **ABI / extension API:** The patch changes src/include/nodes/execnodes.h, which extensions compile against.
- **pg_dump / restore round-trip:** Restore of a dump fails on COPY for a table that holds rows whose virtual column cannot be computed.
- **security / privileges:** DETAIL now shows computed virtual column values, so column privilege rules for the failing-row text apply to them.
- **maintenance burden ("who maintains this in 10 years"):** A write-time computability rule that ALTER TABLE ... SET EXPRESSION and ADD COLUMN do not enforce must be defended for later DDL paths.
- **performance on the hot path:** ExecConstraints() evaluates every virtual column on every insert, update and COPY FROM row, even without constraints.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A STORED column or a check constraint on the virtual column already gives write-time validation to users who want it.

## Who would commit it, and when

- Owner: silent. Someone who knows the executor constraint path and logical replication apply would have to own it. Tom Lane proposed the direction in message #11 and Peter Eisentraut removed himself as committer; neither has commented on v8.
- Priority: needs design rethink; audience niche; review cost medium. The DETAIL improvement helps users of virtual columns with constraints, and the patch is small and limited to execMain.c and execnodes.h. The unconditional write-time computation adds restore and apply failures that nobody has accepted, so the scope needs a decision first.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
