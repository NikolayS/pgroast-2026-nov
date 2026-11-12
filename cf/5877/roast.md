# Design pre-review: Report bytes and transactions actually sent downtream

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5877](https://commitfest.postgresql.org/patch/5877/); [thread](https://www.postgresql.org/message-id/flat/CAExHW5s6KntzUyUoMbKR5dgwRmdV2Ay_2%2BAnTgYGAzo%3DQv61wA%40mail.gmail.com), 123 of 123 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, and the thread shows demand and committer support for one additive counter kept by core. Two fixable level 2 points remain. standards-compat-1 shows that output_bytes counts the WALData header for walsender consumers but drops the LSN and XID columns for SQL API consumers. standards-compat-2 shows that the documented definition does not cover non-transactional logical messages. Both need a committer's decision on the column's definition before commit.

Not checked: The code was not read, so the cost of the increment in OutputPluginWrite() and the exact bytes counted in each output writer are not checked. The refactoring patch that adds replslot_stats.h was not shown and is not judged here.

## Prior attempts

None found before 2025-06-30.

## Objections a reviewer is likely to raise

### 1. One output_bytes column counts different framing per consumer path

An operator has two slots on the same publication. One is read by a walsender and the other by pg_logical_slot_get_changes(). The operator compares output_bytes and sees the walsender slot about 25 bytes higher per message for identical output, then reads that as more data produced.

Message #117 removed the SQL API's LSN and XID columns because they are not plugin output. Message #123 keeps the 25-byte WALData header in the walsender path as output-writer framing. Message #112 accepted the name because protocol-level bytes do not belong in slot stats, yet the WALData header is protocol framing. The column therefore has no single definition, and the documentation can only describe the difference.

**Question:** Why should output_bytes include the WALData header for walsender consumers but exclude the LSN and XID columns for SQL API consumers, when both are framing added outside the plugin?

<sub>fixable, level 2, design; messages [#112](https://www.postgresql.org/message-id/CAD21AoC%2BJoYKBAgTXR4fDWeL71cRDOvx-JmCOVQhWhCHG6r34w%40mail.gmail.com), [#114](https://www.postgresql.org/message-id/CAA4eK1KVQfQtdYKocfsxB1njEQGx4fhMX4N0bjmozG7Sx_V8Sw%40mail.gmail.com), [#117](https://www.postgresql.org/message-id/CAE9k0PnBLCA0ZrzanQVyQCRnfk3rRZqctejCC3Cp9ZnhjMiyvQ%40mail.gmail.com), [#121](https://www.postgresql.org/message-id/802ced07-1530-4cb3-9bf3-215fe61308ee%40gmail.com), [#123](https://www.postgresql.org/message-id/CAE9k0PmtTYcayY5i3oDFazbZy0QqVYYiYqxP9FF-EY8RZcsj_g%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Counting of non-transactional logical messages is not settled in the contract

A user emits non-transactional messages with pg_logical_emit_message(false, ...) and watches output_bytes. The value does not move until a later transaction triggers a stats update. If a later fix changes this behaviour or total_bytes, dashboards built on the first release change meaning without notice.

Message #89 shows that UpdateDecodingStats() does not run for non-transactional messages, so their bytes are not reported when they are sent. Message #91 calls this probably a bug but defers it. Message #114 defines output_bytes as transaction changes plus their delimiting messages and does not cover non-transactional messages. The first release of the column would then document one rule and behave by another.

**Question:** Are non-transactional logical messages counted in output_bytes, when do they show up in the view, and will a later fix to total_bytes change either column's meaning?

<sub>fixable, level 2, design; messages [#89](https://www.postgresql.org/message-id/CAE9k0PmNLURfRzdAXDWZ9-fG6kUjxaa0NCQ3kE8PkATFAKUq4A%40mail.gmail.com), [#91](https://www.postgresql.org/message-id/CAA4eK1LPf_CiBCH0wCv1OHMft1X0m2%3Dytih%2Bva65x3RvPgU7SA%40mail.gmail.com), [#95](https://www.postgresql.org/message-id/CAA4eK1KoOGfvbrX73z_wo8oGHx5Yb9KDJABr%3DJ4cRUay2Vdezw%40mail.gmail.com), [#98](https://www.postgresql.org/message-id/CAE9k0PnmyO62ZhP2yX_9GWeVtOYtXmjLH3oLzEvzec-rsWyVDw%40mail.gmail.com), [#114](https://www.postgresql.org/message-id/CAA4eK1KVQfQtdYKocfsxB1njEQGx4fhMX4N0bjmozG7Sx_V8Sw%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Why does output_bytes include the WALData header for walsender consumers but exclude the LSN and XID columns for SQL API consumers, when the output writer adds both?
2. Are non-transactional logical messages counted in output_bytes, and when do they appear in the view?
3. Will a later fix to the stats update for non-transactional messages change the meaning of output_bytes or total_bytes?
4. How does an operator use output_bytes alone to tell a slow walsender from a slow consumer in the Debezium case from message #1?
5. What is the measured cost of the new counter in OutputPluginWrite() for a workload with many small changes?

## Touch points

- **on-disk format / pg_upgrade:** The pgstat replication slot entry gains a counter, so the stats file format changes.
- **WAL / replication / standby:** The patch touches walsender.c, logical.c and reorderbuffer.c in the logical decoding output path.
- **wire protocol:** see objection standards-compat-1: One output_bytes column counts different framing per consumer path
- **catalog changes:** system_views.sql and pg_proc.dat change to add the output_bytes column to pg_stat_replication_slots.
- **ABI / extension API:** The refactoring patch adds replslot_stats.h and changes structs in pgstat.h and reorderbuffer.h that extensions include.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: One output_bytes column counts different framing per consumer path
- **maintenance burden ("who maintains this in 10 years"):** The counter's meaning depends on what each output writer puts in the buffer before the plugin writes.
- **performance on the hot path:** OutputPluginWrite() increments the counter for every logical decoding output message, and the thread has no cost numbers.

## Who would commit it, and when

- Owner: qualified committer engaged. Masahiko Sawada, Amit Kapila and Michael Paquier accepted the name and scope in the thread; none has replied to the 2026-08-24 version. The owner needs to know logical decoding and the cumulative statistics code.
- Priority: proceed; audience a sizeable minority; review cost medium. Users of logical replication with filtered publications or third-party consumers gain a counter that core maintains, and the thread already agreed on its name and scope. The patch touches walsender, logical decoding and pgstat code, and two definition points still need an answer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
