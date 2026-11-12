# Design pre-review: DDL deparse

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7128](https://commitfest.postgresql.org/patch/7128/) (Needs review); authors: Masahiko Sawada (masahikosawada)
- Thread: [DDL deparse](https://www.postgresql.org/message-id/flat/CAD21AoCwug_gwCfimtoer%3DJsLCnHv8mjqmD7JDXH8K-08jkx-A%40mail.gmail.com), started 2026-06-18
- Messages read: 14 of 14
- Patch prose read: v2-0003-Deparse-ALTER-TABLE.patch, v2-0004-Update-the-DDL-deparse-regression-tests-for-ALTER.patch, v2-0001-Add-a-DDL-deparser-for-CREATE-TABLE.patch, v2-0002-Add-regression-tests-for-DDL-deparse.patch, ddl_deparse.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has lasting demand, so level 1b does not fail, but level 2 does. prior-attempts-1 shows the series keeps the hand-written deparser design that [P1] was withdrawn over, and its replay test only counts declined commands. replication-ondisk-1 and replication-ondisk-2 show the intent-preserving rule lets publisher and subscriber diverge on implicit names and session settings. Confidence is medium because only 3 of 14 messages are visible.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 14 messages and no code were read. Not checked: whether expressions from ruleutils are schema-qualified independently of search_path, and how the deparser handles commands it declines.

## The proposal, restated

**Problem.** An event trigger sees that a DDL command ran but cannot reproduce it elsewhere, because the query string depends on search_path and parse analysis rewrites clauses. The main target is logical replication of DDL; auditing and schema synchronization are also named. The author also asks whether a deparser should preserve user intent or rebuild catalog state, as pg_get_table_ddl() does.

**Proposed design.** The deparser takes the raw parse tree plus the catalogs and builds a self-documenting JSON blob, which ddl_json.c expands back into SQL. It preserves user intent: it omits clauses the user did not write, keeps serial as serial, schema-qualifies names, and does not split one command into several. The ALTER COLUMN TYPE ... USING text is captured during execution, before a sibling DROP COLUMN removes a referenced column.

**User-visible surface.**
- JSON deparse format: "fmt" strings with %{...} placeholders plus named fields, editable before expansion.
- C API: EventTriggerGetCollectedCommands() and EventTriggerCollectAlterColumnTypeUsing() in event_trigger.h; new ddl_deparse.h header.
- Test module test_ddl_deparse with GUCs test_ddl_deparse.print_deparsed_ddl ('json|text|both') and test_ddl_deparse.execute_deparsed_ddl.
- Planned, not in this series: a DDL WAL record and a logical replication protocol message carrying xid, flags, LSN, relid and versioned JSON with the executing role.

**What the author claims.**
- Preserving user intent lets a replay use the defaults of the target, such as default TOAST compression across major versions.
- Deparsing the parse tree recovers information that WAL or catalogs lack, such as the USING expression.
- The v2 series fixes bugs in the earlier patch, including non-executable UNIQUE ... USING INDEX TABLESPACE output and a lost fillfactor.
- 001_deparse_regress.pl replays the core regression suite through the deparser and shows CREATE/ALTER TABLE commands deparse correctly.
- DDL replication needs its own WAL record, because pg_logical_emit_message() is granted to PUBLIC and is unsafe for DDL apply.

**Thread state.** Bharath Rupireddy, Vitaly Davydov and Nikhil Sontakke replied; Vitaly doubts parse-tree deparse and prefers decoding catalog changes from WAL, and Nikhil supports the intent-preserving design and sent fix patches. The author says an updated patch is coming, and the CommitFest entry needs a rebase.

## 2. Prior attempts

### Support logical replication of DDL commands (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/3595/](https://commitfest.postgresql.org/patch/3595/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com)
- **Date:** decided 2024-03-28; started 2022-02-21
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier objected that the hand-written deparser, with its one-to-one mapping of node enums to hardcoded text, makes every change to the parse node structures of replicated DDL also require a change here. He asked for generated code instead, and the authors argued that much of the deparsing cannot be generated. Work stopped after July 2023, and in March 2024 Amit Kapila said the entry should be closed because it still needs a lot of work.
- **Reason (quote):** "As presented, this stuff has an extremely high maintenance cost long-term because it requires anybody doing a change in the parsed tree structures of the DDLs replicated to also change the code paths introduced by this patch set." ([message](https://www.postgresql.org/message-id/ZIgf3qBvFoTsMhIc%40paquier.xyz))
- **What is different this time:** This version keeps user intent, covers only CREATE TABLE and ALTER TABLE, and replays the core regression suite through the deparser. It moves the WAL record and the protocol message to a later series.


## 3. Objections a reviewer is likely to raise

### 1. Hand-written deparser keeps the maintenance cost that stopped the earlier patch

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `prior-attempts-1`

A later patch adds a clause to CREATE TABLE or ALTER TABLE and does not touch ddl_deparse.c. If the change makes the deparser decline the command, 001_deparse_regress.pl only prints a count. If the deparser drops the clause and no later test output depends on it, the replay also passes, and DDL replication users find the missing clause only when the subscriber schema differs.

The earlier DDL replication patch [P1] ended over this cost: every change to the parse node structures of replicated DDL must also change the deparser. This series is still a hand-written mapping from parse nodes to text. The 0002 commit message says declined commands are reported, not asserted, and the 0004 message says the replay cannot check rules that leave the same catalogs. Message #10 calls the detection reliable, but no message answers the [P1] objection or discusses generated code.

**Precedents.**
- [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (Withdrawn, 2024-03-28); thread: [link](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com)

Thread: [#10 (Masahiko Sawada, 2026-08-10)](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com)

**Question:** What makes a grammar or parse-node change fail a test when the patch forgets ddl_deparse.c, including clauses that leave the catalogs unchanged?

Counter-check: held. [P1] concerns the same hand-written design, so the precedent applies. Message #10 calls detection reliable, but the 0002 message says declined commands are only counted and the 0004 message says replay cannot check rules that leave the catalogs the same.

### 2. Implicit object names diverge between publisher and subscriber on replay

fixable · level 2, rubric item 7 · WAL / replication / standby, catalog changes · `replication-ondisk-1`

A subscriber already holds a relation named t_a_seq. `create table t (a serial)` replays as serial, so the subscriber names its sequence t_a_seq1. A later replicated `alter sequence public.t_a_seq ...` then changes the wrong sequence or fails, and apply stops or the schemas drift without notice.

The v2-0001 commit message says a serial column is kept as serial and a clause the user did not write is not emitted. The subscriber then picks the names of implicit objects, such as the sequence and possibly the constraints and indexes. Later deparsed ALTER and DROP commands use the names from the publisher's catalogs, as the identity fields in message #14 show. The proposal does not say how a replayed command and the commands after it agree on these names.

Thread: [#10 (Masahiko Sawada, 2026-08-10)](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#12 (Masahiko Sawada, 2026-10-08)](https://www.postgresql.org/message-id/CAD21AoB5Fsw%3DarTiXjnn1N%2BXS75xcSPNXVMfNVre2LKXh4fgaQ%40mail.gmail.com), [#14 (Masahiko Sawada, 2026-11-12)](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com)

**Question:** How does the subscriber end up with the same names as the publisher for implicit sequences, constraints and indexes when its namespace already differs?

Counter-check: held. Keeping serial as serial and omitting unwritten names lets the subscriber choose implicit names, while later commands carry the publisher's names. No message says how the two agree when the subscriber's namespace already differs.

### 3. Session settings that shape the table are lost on the subscriber

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-2`

A session runs `set default_tablespace = 'fast'` or sets default_table_access_method, then `create table t (a int)`. The deparser omits the clause because the user did not write it. The apply worker runs with its own settings, so the subscriber puts t in another tablespace or access method, and nobody sees the difference until a failover.

In message #5 the author treats a default as replaceable by the target's default, using default_toast_compression as the example. A session SET of the same setting is also an explicit choice, but the rule in v2-0001 cannot tell it apart from no choice at all. The planned envelope in message #14 carries the role but no session settings. Vitaly's question in message #8 about which settings to replicate is still open.

Thread: [#5 (Masahiko Sawada, 2026-06-26)](https://www.postgresql.org/message-id/CAD21AoAkTGcdCHiCOj7_OiT5m4FTCGmO4c1awRdN%3D-6gi6GN_Q%40mail.gmail.com), [#8 (Vitaly Davydov, 2026-06-30)](https://www.postgresql.org/message-id/76843a9e-f2f9-4dbd-93fe-4b3db6ee1a9c%40postgrespro.ru), [#9 (Masahiko Sawada, 2026-06-30)](https://www.postgresql.org/message-id/CAD21AoABOvWC8GTDUYvNzTDoxMJtdUSiNzBOg8kd6ao%2BrEZf%2Bw%40mail.gmail.com), [#14 (Masahiko Sawada, 2026-11-12)](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com)

**Question:** Which session settings does the deparser treat as user intent, and how does the planned envelope bring them to the apply worker?

Counter-check: held. Message #5 covers server-wide defaults only. A session `set` of default_tablespace or default_table_access_method is a user choice that the omit rule drops, and message #9 leaves open which settings count as intent.

### 4. The JSON format becomes a cross-version contract with no stability rules

fixable · level 2, rubric item 6 · wire protocol, SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-2`

A newer publisher emits a fmt string with a field or clause the older subscriber's expander does not know, during a major-version upgrade, which is the main replication use case. The subscriber either fails apply or expands SQL it cannot run. Nobody has defined which side should handle that.

The format already changed between the patch in message #1 and v1 in message #10, with new placeholders and new fields. The "version" field shows up only in the planned envelope in message #14, and the series has no documentation for the format. Because fmt strings carry SQL syntax from the publisher, the expander alone cannot downgrade a command for an older target.

Thread: [#1 (Masahiko Sawada, 2026-06-18)](https://www.postgresql.org/message-id/CAD21AoCwug_gwCfimtoer%3DJsLCnHv8mjqmD7JDXH8K-08jkx-A%40mail.gmail.com), [#10 (Masahiko Sawada, 2026-08-10)](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#14 (Masahiko Sawada, 2026-11-12)](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com)

**Question:** What compatibility rule will the JSON format follow across major versions, and what does an older subscriber do with a field or clause it does not know?

Counter-check: held. Message #14 plans a version field but gives no rule for a field or clause the subscriber does not know. The scenario's direction is wrong for upgrades, where the publisher is older, but replication from a newer to an older server is still unanswered.

### 5. Deparser lands in core with no in-core caller or SQL entry point

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), ABI / extension API · `already-exists-1`

Core ships ddl_deparse.c and ddl_json.c, but only the test module calls them, and DDL replication arrives in a later series or not at all. Every later grammar change to CREATE TABLE or ALTER TABLE must also update the deparser. Users get no feature, and committers carry the upkeep from the first release.

The 0001 commit message says nothing in the tree calls the deparser, and the series adds no SQL function. In message #7 the author agrees that logging, auditing and WAL mining can be handled outside core, which leaves future DDL replication as the only core consumer. The small pieces an extension cannot add itself, EventTriggerGetCollectedCommands() and the USING capture in tablecmds.c, could go in alone so that the deparser can mature as an extension. The in-tree regression replay is a real benefit, but it guards code with no user yet.

**Precedents.**
- [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (Withdrawn, 2024-03-28); thread: [link](https://www.postgresql.org/message-id/flat/CAAD30U+pVmfKwUKy8cbZOnUXyguJ-uBNejwD75Kyo=OjdQGJ9g@mail.gmail.com)

Thread: [#7 (Masahiko Sawada, 2026-06-27)](https://www.postgresql.org/message-id/CAD21AoBCXrzhzLFoT%3DR_qRcBVUG-hqp8QyHOfTMynv6srggiCQ%40mail.gmail.com), [#10 (Masahiko Sawada, 2026-08-10)](https://www.postgresql.org/message-id/CAD21AoDtueBCmmyd2b7rU%3DJd_T1_rX5QAHnRcfpxm%2BSKnCk6hQ%40mail.gmail.com), [#12 (Masahiko Sawada, 2026-10-08)](https://www.postgresql.org/message-id/CAD21AoB5Fsw%3DarTiXjnn1N%2BXS75xcSPNXVMfNVre2LKXh4fgaQ%40mail.gmail.com), [#14 (Masahiko Sawada, 2026-11-12)](https://www.postgresql.org/message-id/CAD21AoC8iYWs6Q4P887vH0kmaN%3D4tNML%2Bxx4DmtQLLR_y5Pc8w%40mail.gmail.com)

**Question:** Why should the deparser enter core before the DDL replication series that consumes it, instead of only the collection hooks while the deparser lives in an extension?

Counter-check: held. Infrastructure-first ordering is accepted practice, but message #12 defers the consumer until the deparser patches settle, and message #7 leaves replication as the only core use. [P1] must not be used as support here, because that patch set included its consumer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-2: Session settings that shape the table are lost on the subscriber |
| WAL / replication / standby | **yes** | Logical replication of DDL is the motivating consumer, and a DDL WAL record is planned for a later series. |
| wire protocol | **yes** | A logical replication protocol message carrying versioned JSON is planned, so the JSON format becomes a cross-version contract. |
| catalog changes | **yes** | see objection replication-ondisk-1: Implicit object names diverge between publisher and subscriber on replay |
| ABI / extension API | **yes** | The series adds EventTriggerGetCollectedCommands(), EventTriggerCollectAlterColumnTypeUsing() and the new ddl_deparse.h header. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: The JSON format becomes a cross-version contract with no stability rules |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The planned envelope carries the executing role, and a subscriber would run the expanded DDL. |
| backpatchability | **yes** | see objection standards-compat-2: The JSON format becomes a cross-version contract with no stability rules |
| maintenance burden ("who maintains this in 10 years") | **yes** | ddl_deparse.c is a hand-written mapping that every later CREATE TABLE or ALTER TABLE syntax change must update. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #7 agrees auditing and logging can live outside core, and no in-tree code calls the deparser yet. |

## 5. Questions the author must answer before anyone reviews code

1. What makes a patch that adds a CREATE TABLE or ALTER TABLE clause fail a test when it does not update ddl_deparse.c, including clauses that leave the catalogs unchanged?
2. How does the subscriber get the same names as the publisher for implicit sequences, constraints and indexes when its namespace already differs?
3. Which session settings, such as default_tablespace and default_table_access_method, does the deparser treat as user intent, and how do they reach the apply worker?
4. What compatibility rule does the JSON format follow across major versions, and what does a subscriber do with a field or clause it does not know?
5. Why should the deparser enter core before the DDL replication series that uses it, instead of only the collection hooks?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The visible thread and CommitFest entry do not identify any participant as a committer. The owner would need to know logical replication, event triggers and the utility command code in tablecmds.c.
- **Liability:** The owner signs for a hand-written deparser that must follow every later change to table DDL syntax, and for a JSON format that becomes a cross-version replication contract once the consumer lands.
- **Author follow-up:** The author posted a v2 series, moved the CommitFest entry forward and says an updated patch is coming. Nikhil Sontakke also sent fix patches.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** DDL replication has lasting demand among logical replication users, but the series is about 250 KiB for two commands and keeps the maintenance design that [P1] was withdrawn over. Code review is premature until the list agrees on the maintenance answer and on the replay semantics for implicit names and session settings.

## Appendix

### Held up in the counter-check, not listed above

- `demand-2` (fixable, level 1b) Auditing and schema sync are named as users but conceded in thread. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-1` Omitting unwritten clauses loses settings that came from session GUCs. Refuted (not specific): Duplicate of replication-ondisk-2, which also ties the gap to the planned envelope and to the open question in message #8.
- `standards-compat-3` No user-callable interface, yet auditing and schema sync are named. Refuted (not specific): Duplicate of demand-2. Both say the 0001 commit message names auditing and schema synchronization, while message #7 concedes those uses and the series has no SQL-callable function.
- `maintenance-fit-1` Hand-written deparser must track every future CREATE/ALTER TABLE change. Refuted (not specific): Duplicate of prior-attempts-1, which states the same maintenance concern with the same evidence from the 0002 and 0004 commit messages.
- `maintenance-fit-2` A second table-DDL renderer next to pg_get_table_ddl and pg_dump. Refuted (already handled): Messages #3 and #7 explain that pg_get_table_ddl() cannot tell written clauses from defaults and does not cover ALTER TABLE, and the 0001 message says expressions and constraints come from ruleutils. The remaining risk of a forgotten deparser update is prior-attempts-1.
- `maintenance-fit-3` Deparser lands with no in-tree consumer and an unstated JSON contract. Refuted (not specific): Duplicate of already-exists-1 for the missing consumer and of standards-compat-2 for the format contract. The extension-breakage scenario is weak, because Postgres does not promise a stable C API across major versions.
- `demand-1` Deparser lands with no consumer while its upkeep starts at once. Refuted (not specific): Duplicate of already-exists-1. Its use of [P1] also does not fit, because that patch set included WAL logging and apply and so had a consumer.

### Review angles

- prior-attempts: 1 objection(s). One objection only. [P7] is about logical WAL records for table AMs and does not fit. Message #1 points to two earlier deparse threads, but the visible material does not report why they stopped.
- standards-compat: 3 objection(s). Three objections raised. Overlap with pg_get_table_ddl() was discussed in messages #1 and #7 and is left to other angles.
- replication-ondisk: 2 objection(s). The series adds no WAL record, catalog change, on-disk format or protocol message. Those are only planned, so these objections are about the replication semantics the intent-preserving design commits to, and the planned record cannot be checked yet.
- security: 0 objection(s). The series adds no privilege surface. It has no SQL-callable function and no new catalog or GUC outside the test module, and it runs only inside event triggers, which need superuser. Its catalog reads stay within what the trigger owner can already see. Security concerns belong to the planned DDL WAL record and protocol message, which are not in this series. One open point there is how a subscriber maps the "role" field in the JSON, a role name taken from the publisher, to its own privileges. Message #14 already rejects pg_logical_emit_message() because it is granted to PUBLIC. One point needs the code: whether expressions taken from ruleutils are schema-qualified independently of the session search_path.
- maintenance-fit: 3 objection(s). Objections raised; see list.
- already-exists: 1 objection(s). pg_get_table_ddl() is still a proposal, so it is not an existing alternative, and the author explains why catalog-state output cannot preserve user intent.
- demand: 2 objection(s). Demand for DDL replication itself is real and has lasted: several prior attempts, plus engaged reviewers from more than one company. The objections concern this series shipping the deparser ahead of that consumer.

### Precedents retrieved

- P1: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - used in the review; reason summarised from its thread
- P2: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P3: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant; reason summarised from its thread
- P4: [Revert recent SQL/JSON related commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4d7e24e0f4d05b546228488ccdc2848a80245ffb) (revert, Reverted, 2024-09-06) - judged not relevant
- P5: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P6: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant; reason summarised from its thread
- P7: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - used in the review; reason summarised from its thread
- P8: [Xact event callback at command start](https://commitfest.postgresql.org/patch/2860/) (cf, Rejected, 2021-09-01) - judged not relevant; reason summarised from its thread
- P9: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant; reason summarised from its thread
- P10: [Time-delayed logical replication subscriber](https://commitfest.postgresql.org/patch/3581/) (cf, Returned with feedback, 2023-03-10) - judged not relevant
- P11: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant
- P12: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant
- P13: [Add pg_get_owned_sequence function](https://commitfest.postgresql.org/patch/4535/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P14: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P15: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/3556/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P16: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P17: [Add a function to get the version of installed extension](https://commitfest.postgresql.org/patch/5514/) (cf, Withdrawn, 2025-04-03) - judged not relevant
- P18: [GET DIAGNOSTICS FUNCTION_NAME](https://commitfest.postgresql.org/patch/1457/) (cf, Rejected, 2018-03-07) - judged not relevant
- P19: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/4636/) (cf, Withdrawn, 2023-11-01) - judged not relevant
- P20: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P21: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P22: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P23: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant
- P24: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 167 s; cost reported by the CLI: $2.90
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 110 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:22:45.385Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
