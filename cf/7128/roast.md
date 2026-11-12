# Design pre-review: DDL deparse

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7128](https://commitfest.postgresql.org/patch/7128/); [thread](https://www.postgresql.org/message-id/flat/CAD21AoCwug_gwCfimtoer%3DJsLCnHv8mjqmD7JDXH8K-08jkx-A%40mail.gmail.com), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has lasting demand, so level 1b does not fail, but level 2 does. prior-attempts-1 shows the series keeps the hand-written deparser design that [P1] was withdrawn over, and its replay test only counts declined commands. replication-ondisk-1 and replication-ondisk-2 show the intent-preserving rule lets publisher and subscriber diverge on implicit names and session settings. Confidence is medium because only 3 of 14 messages are visible.

Not checked: Only 3 of 14 messages and no code were read. Not checked: whether expressions from ruleutils are schema-qualified independently of search_path, and how the deparser handles commands it declines.

## Prior attempts

- [Support logical replication of DDL commands](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com) (same idea; Withdrawn, 2024-03-28). "As presented, this stuff has an extremely high maintenance cost long-term because it requires anybody doing a change in the parsed tree structures of the DDLs replicated to also change the code paths introduced by this patch set." ([message](https://www.postgresql.org/message-id/ZIgf3qBvFoTsMhIc%40paquier.xyz)) Different now: This version keeps user intent, covers only CREATE TABLE and ALTER TABLE, and replays the core regression suite through the deparser. It moves the WAL record and the protocol message to a later series.

## Objections a reviewer is likely to raise

### 1. Hand-written deparser keeps the maintenance cost that stopped the earlier patch

A later patch adds a clause to CREATE TABLE or ALTER TABLE and does not touch ddl_deparse.c. If the change makes the deparser decline the command, 001_deparse_regress.pl only prints a count. If the deparser drops the clause and no later test output depends on it, the replay also passes, and DDL replication users find the missing clause only when the subscriber schema differs.

The earlier DDL replication patch [P1] ended over this cost: every change to the parse node structures of replicated DDL must also change the deparser. This series is still a hand-written mapping from parse nodes to text. The 0002 commit message says declined commands are reported, not asserted, and the 0004 message says the replay cannot check rules that leave the same catalogs. Message #10 calls the detection reliable, but no message answers the [P1] objection or discusses generated code.

Precedent: [Support logical replication of DDL commands](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com) (Withdrawn, 2024-03-28)

**Question:** What makes a grammar or parse-node change fail a test when the patch forgets ddl_deparse.c, including clauses that leave the catalogs unchanged?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#10](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Implicit object names diverge between publisher and subscriber on replay

A subscriber already holds a relation named t_a_seq. `create table t (a serial)` replays as serial, so the subscriber names its sequence t_a_seq1. A later replicated `alter sequence public.t_a_seq ...` then changes the wrong sequence or fails, and apply stops or the schemas drift without notice.

The v2-0001 commit message says a serial column is kept as serial and a clause the user did not write is not emitted. The subscriber then picks the names of implicit objects, such as the sequence and possibly the constraints and indexes. Later deparsed ALTER and DROP commands use the names from the publisher's catalogs, as the identity fields in message #14 show. The proposal does not say how a replayed command and the commands after it agree on these names.

**Question:** How does the subscriber end up with the same names as the publisher for implicit sequences, constraints and indexes when its namespace already differs?

<sub>fixable, level 2, WAL / replication / standby; messages [#10](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAD21AoB5Fsw%3DarTiXjnn1N%2BXS75xcSPNXVMfNVre2LKXh4fgaQ%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Session settings that shape the table are lost on the subscriber

A session runs `set default_tablespace = 'fast'` or sets default_table_access_method, then `create table t (a int)`. The deparser omits the clause because the user did not write it. The apply worker runs with its own settings, so the subscriber puts t in another tablespace or access method, and nobody sees the difference until a failover.

In message #5 the author treats a default as replaceable by the target's default, using default_toast_compression as the example. A session SET of the same setting is also an explicit choice, but the rule in v2-0001 cannot tell it apart from no choice at all. The planned envelope in message #14 carries the role but no session settings. Vitaly's question in message #8 about which settings to replicate is still open.

**Question:** Which session settings does the deparser treat as user intent, and how does the planned envelope bring them to the apply worker?

<sub>fixable, level 2, WAL / replication / standby; messages [#5](https://www.postgresql.org/message-id/CAD21AoAkTGcdCHiCOj7_OiT5m4FTCGmO4c1awRdN%3D-6gi6GN_Q%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/76843a9e-f2f9-4dbd-93fe-4b3db6ee1a9c%40postgrespro.ru), [#9](https://www.postgresql.org/message-id/CAD21AoABOvWC8GTDUYvNzTDoxMJtdUSiNzBOg8kd6ao%2BrEZf%2Bw%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com); `replication-ondisk-2`</sub>

### 4. The JSON format becomes a cross-version contract with no stability rules

A newer publisher emits a fmt string with a field or clause the older subscriber's expander does not know, during a major-version upgrade, which is the main replication use case. The subscriber either fails apply or expands SQL it cannot run. Nobody has defined which side should handle that.

The format already changed between the patch in message #1 and v1 in message #10, with new placeholders and new fields. The "version" field shows up only in the planned envelope in message #14, and the series has no documentation for the format. Because fmt strings carry SQL syntax from the publisher, the expander alone cannot downgrade a command for an older target.

**Question:** What compatibility rule will the JSON format follow across major versions, and what does an older subscriber do with a field or clause it does not know?

<sub>fixable, level 2, wire protocol; messages [#1](https://www.postgresql.org/message-id/CAD21AoCwug_gwCfimtoer%3DJsLCnHv8mjqmD7JDXH8K-08jkx-A%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com); `standards-compat-2`</sub>

### 5. Deparser lands in core with no in-core caller or SQL entry point

Core ships ddl_deparse.c and ddl_json.c, but only the test module calls them, and DDL replication arrives in a later series or not at all. Every later grammar change to CREATE TABLE or ALTER TABLE must also update the deparser. Users get no feature, and committers carry the upkeep from the first release.

The 0001 commit message says nothing in the tree calls the deparser, and the series adds no SQL function. In message #7 the author agrees that logging, auditing and WAL mining can be handled outside core, which leaves future DDL replication as the only core consumer. The small pieces an extension cannot add itself, EventTriggerGetCollectedCommands() and the USING capture in tablecmds.c, could go in alone so that the deparser can mature as an extension. The in-tree regression replay is a real benefit, but it guards code with no user yet.

Precedent: [Support logical replication of DDL commands](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com) (Withdrawn, 2024-03-28)

**Question:** Why should the deparser enter core before the DDL replication series that consumes it, instead of only the collection hooks while the deparser lives in an extension?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#7](https://www.postgresql.org/message-id/CAD21AoBCXrzhzLFoT%3DR_qRcBVUG-hqp8QyHOfTMynv6srggiCQ%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAD21AoB5Fsw%3DarTiXjnn1N%2BXS75xcSPNXVMfNVre2LKXh4fgaQ%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. What makes a patch that adds a CREATE TABLE or ALTER TABLE clause fail a test when it does not update ddl_deparse.c, including clauses that leave the catalogs unchanged?
2. How does the subscriber get the same names as the publisher for implicit sequences, constraints and indexes when its namespace already differs?
3. Which session settings, such as default_tablespace and default_table_access_method, does the deparser treat as user intent, and how do they reach the apply worker?
4. What compatibility rule does the JSON format follow across major versions, and what does a subscriber do with a field or clause it does not know?
5. Why should the deparser enter core before the DDL replication series that uses it, instead of only the collection hooks?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-2: Session settings that shape the table are lost on the subscriber
- **WAL / replication / standby:** Logical replication of DDL is the motivating consumer, and a DDL WAL record is planned for a later series.
- **wire protocol:** A logical replication protocol message carrying versioned JSON is planned, so the JSON format becomes a cross-version contract.
- **catalog changes:** see objection replication-ondisk-1: Implicit object names diverge between publisher and subscriber on replay
- **ABI / extension API:** The series adds EventTriggerGetCollectedCommands(), EventTriggerCollectAlterColumnTypeUsing() and the new ddl_deparse.h header.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: The JSON format becomes a cross-version contract with no stability rules
- **security / privileges:** The planned envelope carries the executing role, and a subscriber would run the expanded DDL.
- **backpatchability:** see objection standards-compat-2: The JSON format becomes a cross-version contract with no stability rules
- **maintenance burden ("who maintains this in 10 years"):** ddl_deparse.c is a hand-written mapping that every later CREATE TABLE or ALTER TABLE syntax change must update.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #7 agrees auditing and logging can live outside core, and no in-tree code calls the deparser yet.

## Who would commit it, and when

- Owner: silent. The visible thread and CommitFest entry do not identify any participant as a committer. The owner would need to know logical replication, event triggers and the utility command code in tablecmds.c.
- Priority: fix design first; audience a sizeable minority; review cost high. DDL replication has lasting demand among logical replication users, but the series is about 250 KiB for two commands and keeps the maintenance design that [P1] was withdrawn over. Code review is premature until the list agrees on the maintenance answer and on the replay semantics for implicit names and session settings.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
