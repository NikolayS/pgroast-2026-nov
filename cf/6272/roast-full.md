# Design pre-review: PoC: Simplify recovery after dropping a table by LOGGING the restore LSN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6272](https://commitfest.postgresql.org/patch/6272/) (Needs review); authors: Nikolay Shaplov (nataraj), Andrey Borodin (x4m), Kirk Wolak (kirkw), Dmitry Lebedev (dmitrylebedev), Salma El-Sayed (salmae)
- Thread: [PoC: Simplify recovery after dropping a table by LOGGING the restore LSN](https://www.postgresql.org/message-id/flat/CACLU5mSr%2Bt%2B6K_Ww6z1fk_%3DjP%3D5-U0CEqMsHjii8uG_6UWQo-g%40mail.gmail.com), started 2024-11-08
- Messages read: 22 of 22
- Patch prose read: vPoC-0001-Simplify-recovery-after-dropping-a-table-by-giv.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support from reviewers, but the case for core code is not yet made. [P7] proposed the same setting for DROP TABLE and TRUNCATE and was rejected because event triggers can do it. already-exists-1 shows that an sql_drop event trigger logging the xid gives an exact target through recovery_target_xid. maintenance-fit-1 shows that the thread does not answer the extension question for v13.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check whether the logging in RecordTransactionCommit can raise an ERROR, or which GUC context v13 uses. Four level 2 objections survived (standards-compat-1, standards-compat-2, replication-ondisk-1, security-1) but are not ranked because level 1b fails first.

## The proposal, restated

**Problem.** A user who drops or truncates the wrong table must find the LSN just before the drop to do point-in-time recovery. The authors call this a common support request. Finding that LSN by hand is slow and error-prone.

**Proposed design.** With a new GUC enabled, the server writes a LOG line with the commit LSN for each committed permanent table drop or truncate, with a hint to use recovery_target_lsn and recovery_target_inclusive = false. Since v13 the line is written in RecordTransactionCommit right after the commit record. Rolled-back work, temporary and unlogged tables, prepared transactions and wal_level = minimal produce no line.

**User-visible surface.**
- GUC log_object_drops (default off); the reviewer says the name may change.
- Server LOG message per table, e.g. table "public.t" (OID 16384) dropped at LSN X/X, with a recovery hint.
- Documentation of the GUC, including the prepared transaction and wal_level = minimal exclusions.

**What the author claims.**
- Accidental table drops are an all too common support request.
- Recovering to the logged LSN with recovery_target_inclusive = false brings the table back.
- An extension cannot easily do this, because the object access hook runs before commit, when the commit LSN does not exist yet.
- The message is logged even when a backend is killed while waiting for synchronous replication after commit.

**Thread state.** Jim Jones, Zsolt Parragi, Kirill Reshke and Andrey Borodin reviewed; DROP DATABASE was removed from scope by agreement, and v13 is the latest. Open points are the GUC name, the missing database name in the default log prefix, and a possible TAP test for the SyncRep case.

## 2. Prior attempts

### automatic restore point (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/1788/](https://commitfest.postgresql.org/patch/1788/)
- **Thread:** [https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23)
- **Date:** decided 2018-09-30; started 2018-06-26
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers said the same result is possible without core changes, through event triggers or the utility hook, and that a hardcoded set of commands is arbitrary and hard to extend. Michael Paquier also called the conditions hard to define and the gain small next to backups and delayed standbys. He rejected the patch because it brings nothing that event triggers cannot do.
- **Reason (quote):** "Right now, you just have it hardcoded to TRUNCATE and DROP TABLE, which seems somewhat arbitrary. With event triggers, you have the full flexibility to do what you want." ([message](https://www.postgresql.org/message-id/d6ca5c16-ab9c-658d-2b4a-618f970ce4b2%402ndquadrant.com))
- **What is different this time:** This proposal logs the exact commit LSN from RecordTransactionCommit and skips rolled-back, temporary, unlogged and two-phase cases. P7 logged an LSN, or created a restore point, before the command ran.


## 3. Objections a reviewer is likely to raise

### 1. Event triggers with recovery_target_xid already give an exact recovery target

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA creates an sql_drop event trigger that does `raise log` with pg_current_xact_id() and the dropped object names. After an accidental DROP TABLE, the DBA sets recovery_target_xid to the logged xid with recovery_target_inclusive = false, and recovery stops just before that commit. Core would then carry a second, narrower mechanism and a new GUC for the same job.

Message #6 says an extension cannot log the commit LSN before commit. That holds for an LSN, but recovery_target_xid needs no commit LSN, and the xid is known when the drop runs. [P7] was rejected for this reason: event triggers already do it, and a fixed list of commands is arbitrary. Event triggers do not fire on TRUNCATE, so the real gap is TRUNCATE and convenience, and the thread argues neither.

**Precedents.**
- [automatic restore point](https://commitfest.postgresql.org/patch/1788/) (Rejected, 2018-09-30); thread: [link](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23)

Thread: [#3 (Kirill Reshke, 2025-03-13)](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com), [#6 (Salma El-Sayed, 2026-09-28)](https://www.postgresql.org/message-id/CANBEAPEMdptqFRpWimDO2TqYLaRr52Ljd_mOrR%2BDGM3VYhw39A%40mail.gmail.com)

**Question:** What does the commit-LSN log line give that an sql_drop event trigger logging the xid for recovery_target_xid cannot, apart from TRUNCATE coverage?

Counter-check: held. recovery_target_xid with recovery_target_inclusive = false stops just before that commit, and the xid is known when sql_drop fires. The thread's answer in message #6 covers only LSN targets, so the remaining gap is TRUNCATE and convenience, as the objection says.

### 2. Earlier rejection on extension grounds not yet answered for v13

fixable · level 1b, rubric item 2 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer reads [P7], which was rejected because event triggers could do the same, and asks why core must own this. The thread gives no answer for the current design. The patch stalls on the same question that ended [P7].

Message #3 asks why an extension cannot do this, and message #6 answers that an extension would need an xact callback plus savepoint tracking. Extensions can already do both. Moving the logging into RecordTransactionCommit (message #21) creates a real gap for the SyncRep kill case, but the thread never states that gap as the reason core is needed. [P7] recorded the same objection for the same pair of commands.

**Precedents.**
- [automatic restore point](https://commitfest.postgresql.org/patch/1788/) (Rejected, 2018-09-30); thread: [link](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23)

Thread: [#3 (Kirill Reshke, 2025-03-13)](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com), [#6 (Salma El-Sayed, 2026-09-28)](https://www.postgresql.org/message-id/CANBEAPEMdptqFRpWimDO2TqYLaRr52Ljd_mOrR%2BDGM3VYhw39A%40mail.gmail.com), [#18 (Salma Elsayed, 2026-10-30)](https://www.postgresql.org/message-id/CANBEAPH36ERVXT1VOWCj8sxTR6vtERVomcyx3vFhZ-wauaK-cw%40mail.gmail.com), [#21 (Salma Elsayed, 2026-11-06)](https://www.postgresql.org/message-id/CANBEAPFzwpUcDcSyN%3DP2Mg7kqTP3Jx8YkPAVMOR2aK%2BXeM0ZKw%40mail.gmail.com)

**Question:** Why is the SyncRep kill window from message #20 serious enough to need core code, and could a small generic post-commit hook close it for an extension instead?

Counter-check: held. Message #6 lists what an extension would need, and extensions already have object access hooks, xact callbacks and the commit LSN. The thread never states the SyncRep kill case as the reason for core code, and Kirill Reshke's question has no answer for v13.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The logging runs in RecordTransactionCommit after the commit record and before the SyncRep wait, and the line gives a recovery target. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds log_object_drops, and reviewers already question whether the name fits the scope. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The proposal does not state the GUC context, so an ordinary session may be able to turn the logging off. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | New code in the commit path must stay error-free, and the covered command set may grow. |
| performance on the hot path | **yes** | With the GUC on, the commit path writes one LOG line per dropped or truncated table. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | [P7] was rejected because event triggers can do the same, and message #3 asks the extension question again. |

## 5. Questions the author must answer before anyone reviews code

1. What does the commit-LSN log line give that an sql_drop event trigger logging the xid for recovery_target_xid cannot, apart from TRUNCATE coverage?
2. Is the SyncRep kill case from message #20 the reason core code is needed, and could a generic post-commit hook let an extension cover it?
3. What guarantees that the per-table logging after the commit record cannot raise an ERROR when one transaction drops thousands of tables?
4. What name and value type will the setting have, and what rule decides which commands it covers?
5. Which GUC context does log_object_drops use, and will the message include the database name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the commit path in xact.c and synchronous replication would have to own it. The visible thread and the CommitFest entry show reviewers but no such committer.
- **Liability:** The owner signs for new code between the commit record and the end of the transaction, where an ERROR becomes a PANIC, for as long as the GUC exists. The owner also signs for the GUC name and scope.
- **Author follow-up:** The patch reached v13 over two years, and an author moved the entry to a new CommitFest in October 2026, so the authors appear to stay with it.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Sites that run PITR would benefit, and reviewers call the feature useful. The core-versus-event-trigger question from [P7] and the GUC name, scope and context need answers before code review; no patch code was read.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Log line does not identify the database of the dropped table. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) GUC name and scope fix an arbitrary command set into the contract. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Log call placed between commit record and ProcArray end can turn errors into PANIC. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-1` (fixable, level 2) Proposal does not say who may set log_object_drops. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Earlier rejection reason, that event triggers suffice, is only partly answered. Refuted (not specific): Duplicate of already-exists-1, which makes the same event-trigger point with an exact target (the xid) and also names the TRUNCATE gap. This version rests on an inexact insert LSN and does not note that event triggers do not fire on TRUNCATE.
- `prior-attempts-2` The covered command set keeps the shape P7 called arbitrary. Refuted (not specific): Duplicate of standards-compat-2, which covers the same scope question and adds the concrete point about the setting's name and value type.
- `standards-compat-3` Privilege level of log_object_drops is not stated. Refuted (not specific): Duplicate of security-1, which cites the session-level SET examples in the thread. The log-flooding addition applies only when the setting is on, and a role that can create and drop tables can already fill the log through other log_* settings.
- `replication-ondisk-2` Logged LSN can point to WAL that is lost after failover. Refuted (no failure scenario): In the scenario the promoted standby never replayed the drop, so the table still exists there and no recovery to that LSN is needed. LSNs without a timeline are existing practice for recovery_target_lsn and for other server log lines such as checkpoint messages.
- `maintenance-fit-2` Fixed command list invites a new special case per release. Refuted (not specific): Duplicate of standards-compat-2. The claim that later patches will add commands to the commit path is a guess; the thread removed DROP DATABASE from scope and message #3 advised against DELETE and UPDATE.
- `maintenance-fit-3` Per-table logging inside RecordTransactionCommit before the SyncRep wait. Refuted (no failure scenario): The PANIC part is a duplicate of replication-ondisk-1. The latency cost falls only on installations that turn the setting on, and the locks held longer are on tables that the same transaction is dropping.
- `demand-1` Demand rests on the authors' support experience, and a 2018 version was rejected. Refuted (already handled): The material shows demand that lasted: an independent author proposed the same feature in 2018 ([P7]), and reviewers in messages #2 and #3 call it useful. The event-trigger part is a duplicate of already-exists-1.
- `demand-2` Default-off GUC misses most users who drop the wrong table. Refuted (not specific): This applies to every default-off logging setting, and PITR itself needs WAL archiving set up in advance. Setting one GUC cluster-wide is a lower bar than an event trigger in each database, and event triggers do not cover TRUNCATE.

### Review angles

- prior-attempts: 2 objection(s). P5 shares only the drop-at-commit area. Its reason concerns removing storage through xact callbacks and does not apply to a log line.
- standards-compat: 3 objection(s). Three objections are listed. None is fatal from the standards and compatibility angle.
- replication-ondisk: 2 objection(s). The proposal adds no WAL record, catalog, on-disk format or protocol change, so pg_upgrade and the wire protocol are not affected. The two objections concern only the commit path and how the logged LSN behaves under replication.
- security: 1 objection(s). The log line reports only drops the session itself committed, and it is filtered to permanent tables, so it leaks no other users' data. The only security point found is the unstated GUC context.
- maintenance-fit: 3 objection(s). Objections were raised. The thread has active reviewers, but the visible material does not name a committer who would own the commit-path change.
- already-exists: 1 objection(s). One objection. pg_waldump can also find the commit record of the drop by hand, but the brief already treats that way as slow.
- demand: 2 objection(s). Two objections raised. Reviewers in messages #2, #3 and #15 call the feature useful, but no independent user demand or field numbers appear in the material.

### Precedents retrieved

- P1: [Improve recovery performance of dropping database having many tablespaces](https://commitfest.postgresql.org/patch/1658/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P2: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P3: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - judged not relevant; reason summarised from its thread
- P4: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P5: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P6: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant; reason summarised from its thread
- P7: [automatic restore point](https://commitfest.postgresql.org/patch/1788/) (cf, Rejected, 2018-09-30) - used in the review; reason summarised from its thread
- P8: [Add last commit LSN to pg_last_committed_xact()](https://commitfest.postgresql.org/patch/3515/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P9: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P10: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P11: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P12: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant
- P13: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P14: [Call RelationDropStorage() for broader range of object drops.](https://commitfest.postgresql.org/patch/1295/) (cf, Rejected, 2018-01-22) - judged not relevant
- P15: [Restore lock level to update statusFlags](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcfff74fb16622898a9d1df33e530f477caab095) (revert, Partially reverted, 2020-11-26) - judged not relevant
- P16: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P17: [Revert "Add GUC backtrace_on_internal_error"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=592a2283721f7143999364ef487f2b4993f5161d) (revert, Reverted, 2024-04-29) - judged not relevant
- P18: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P19: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P20: [Update relfrozenxmin when truncating temp tables](https://commitfest.postgresql.org/patch/3358/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P21: [Add a GUC variable that control logical replication](https://commitfest.postgresql.org/patch/2296/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P22: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P23: [Logical decoding of two-phase transactions](https://commitfest.postgresql.org/patch/944/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P24: [Automatic HASH and LIST partition creation](https://commitfest.postgresql.org/patch/2694/) (cf, Returned with feedback, 2021-12-02) - judged not relevant

### Warnings

- review angle "already-exists" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 205 s; cost reported by the CLI: $2.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 121 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:57.959Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
