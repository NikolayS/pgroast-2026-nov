# Design pre-review: PoC: Simplify recovery after dropping a table by LOGGING the restore LSN

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6272](https://commitfest.postgresql.org/patch/6272/); [thread](https://www.postgresql.org/message-id/flat/CACLU5mSr%2Bt%2B6K_Ww6z1fk_%3DjP%3D5-U0CEqMsHjii8uG_6UWQo-g%40mail.gmail.com), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support from reviewers, but the case for core code is not yet made. [P7] proposed the same setting for DROP TABLE and TRUNCATE and was rejected because event triggers can do it. already-exists-1 shows that an sql_drop event trigger logging the xid gives an exact target through recovery_target_xid. maintenance-fit-1 shows that the thread does not answer the extension question for v13.

Not checked: Without the code, the review could not check whether the logging in RecordTransactionCommit can raise an ERROR, or which GUC context v13 uses. Four level 2 objections survived (standards-compat-1, standards-compat-2, replication-ondisk-1, security-1) but are not ranked because level 1b fails first.

## Prior attempts

- [automatic restore point](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23) (same idea; Rejected, 2018-09-30). "Right now, you just have it hardcoded to TRUNCATE and DROP TABLE, which seems somewhat arbitrary. With event triggers, you have the full flexibility to do what you want." ([message](https://www.postgresql.org/message-id/d6ca5c16-ab9c-658d-2b4a-618f970ce4b2%402ndquadrant.com)) Different now: This proposal logs the exact commit LSN from RecordTransactionCommit and skips rolled-back, temporary, unlogged and two-phase cases. P7 logged an LSN, or created a restore point, before the command ran.

## Objections a reviewer is likely to raise

### 1. Event triggers with recovery_target_xid already give an exact recovery target

A DBA creates an sql_drop event trigger that does `raise log` with pg_current_xact_id() and the dropped object names. After an accidental DROP TABLE, the DBA sets recovery_target_xid to the logged xid with recovery_target_inclusive = false, and recovery stops just before that commit. Core would then carry a second, narrower mechanism and a new GUC for the same job.

Message #6 says an extension cannot log the commit LSN before commit. That holds for an LSN, but recovery_target_xid needs no commit LSN, and the xid is known when the drop runs. [P7] was rejected for this reason: event triggers already do it, and a fixed list of commands is arbitrary. Event triggers do not fire on TRUNCATE, so the real gap is TRUNCATE and convenience, and the thread argues neither.

Precedent: [automatic restore point](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23) (Rejected, 2018-09-30)

**Question:** What does the commit-LSN log line give that an sql_drop event trigger logging the xid for recovery_target_xid cannot, apart from TRUNCATE coverage?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#3](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CANBEAPEMdptqFRpWimDO2TqYLaRr52Ljd_mOrR%2BDGM3VYhw39A%40mail.gmail.com); `already-exists-1`</sub>

### 2. Earlier rejection on extension grounds not yet answered for v13

A committer reads [P7], which was rejected because event triggers could do the same, and asks why core must own this. The thread gives no answer for the current design. The patch stalls on the same question that ended [P7].

Message #3 asks why an extension cannot do this, and message #6 answers that an extension would need an xact callback plus savepoint tracking. Extensions can already do both. Moving the logging into RecordTransactionCommit (message #21) creates a real gap for the SyncRep kill case, but the thread never states that gap as the reason core is needed. [P7] recorded the same objection for the same pair of commands.

Precedent: [automatic restore point](https://www.postgresql.org/message-id/flat/8E9126CB6CE2CD42962059AB0FBF7B0DBE3167@g01jpexmbkw23) (Rejected, 2018-09-30)

**Question:** Why is the SyncRep kill window from message #20 serious enough to need core code, and could a small generic post-commit hook close it for an extension instead?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#3](https://www.postgresql.org/message-id/CALdSSPiRDvbuPgZKAB1%2BBDb3Nfe-i3PYkiy%3DScMZ7tU5f0toKQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CANBEAPEMdptqFRpWimDO2TqYLaRr52Ljd_mOrR%2BDGM3VYhw39A%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CANBEAPH36ERVXT1VOWCj8sxTR6vtERVomcyx3vFhZ-wauaK-cw%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CANBEAPFzwpUcDcSyN%3DP2Mg7kqTP3Jx8YkPAVMOR2aK%2BXeM0ZKw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What does the commit-LSN log line give that an sql_drop event trigger logging the xid for recovery_target_xid cannot, apart from TRUNCATE coverage?
2. Is the SyncRep kill case from message #20 the reason core code is needed, and could a generic post-commit hook let an extension cover it?
3. What guarantees that the per-table logging after the commit record cannot raise an ERROR when one transaction drops thousands of tables?
4. What name and value type will the setting have, and what rule decides which commands it covers?
5. Which GUC context does log_object_drops use, and will the message include the database name?

## Touch points

- **WAL / replication / standby:** The logging runs in RecordTransactionCommit after the commit record and before the SyncRep wait, and the line gives a recovery target.
- **new GUC:** The patch adds log_object_drops, and reviewers already question whether the name fits the scope.
- **security / privileges:** The proposal does not state the GUC context, so an ordinary session may be able to turn the logging off.
- **maintenance burden ("who maintains this in 10 years"):** New code in the commit path must stay error-free, and the covered command set may grow.
- **performance on the hot path:** With the GUC on, the commit path writes one LOG line per dropped or truncated table.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** [P7] was rejected because event triggers can do the same, and message #3 asks the extension question again.

## Who would commit it, and when

- Owner: silent. Someone who knows the commit path in xact.c and synchronous replication would have to own it. The visible thread and the CommitFest entry show reviewers but no such committer.
- Priority: fix design first; audience a sizeable minority; review cost medium. Sites that run PITR would benefit, and reviewers call the feature useful. The core-versus-event-trigger question from [P7] and the GUC name, scope and context need answers before code review; no patch code was read.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
