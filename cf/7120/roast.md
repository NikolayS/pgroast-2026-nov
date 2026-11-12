# Design pre-review: Small fixes needed by high-availability tools

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7120](https://commitfest.postgresql.org/patch/7120/); [thread](https://www.postgresql.org/message-id/flat/0B44E464-BA62-4056-9465-3320DD2D0302%40yandex-team.ru), 25 of 25 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Demand is shown and both settings are off by default, so level 1 does not stop the proposal. The design is still open at level 2. Session termination still releases a non-replicated commit on a running primary (replication-ondisk-1), and the author's own question of GUC, hook or both has no reply (maintenance-fit-1). The WAL sender exemption (security-1) and the reason the gate must live in core (already-exists-1) also need an answer before code review.

Not checked: Only 3 of 25 messages and no code were read. Whether v3 limits the WAL sender exemption to physical senders, and how the per-level shared state behaves on promotion, can only be checked in the code.

## Prior attempts

- [GUC to disable cancellation of awaiting for synchronous replication](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru) (same idea; Returned with feedback, 2021-10-06). "I don't think that preventing a backend from being canceled during waiting for sync rep actually addresses your issue." ([message](https://www.postgresql.org/message-id/4f8d54c9-6f18-23d5-c4de-9d6656d3a408%40oss.nttdata.com)) Different now: v3 pairs the cancel GUC with a post-recovery connection gate, which covers the restart and crash case that reviewers raised there.
- [GUC to prevent cancellation while waiting for sync rep](https://www.postgresql.org/message-id/flat/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz) (similar idea; Withdrawn, 2021-07-13). "OK. I don't think ignoring SIGTERM in the way my patch does it is a great solution, and it's not getting much support, so I think I'll back away from that idea." ([message](https://www.postgresql.org/message-id/c162f1d08a04c78c5127b37fcfc02527900a8029.camel%40j-davis.com)) Different now: v3 ignores only query cancel and still honors session termination and shutdown, so it avoids the SIGTERM objection that led to that withdrawal.
- [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://www.postgresql.org/message-id/flat/CALj2ACUrOB59QaE6=jF2cFAyv1MR7fzD8tr4YM5+OwEYG1SNzA@mail.gmail.com) (similar idea; Returned with feedback, 2023-08-02). "Since there hasn't been any updates for the past six months, and the patch hasn't applied for a few months, I am marking this returned with feedback for now." ([message](https://www.postgresql.org/message-id/D52A9851-E4C5-40CD-98E5-F6BC0417ED0F%40yesql.se)) Different now: v3 blocks cancel for the whole wait instead of a naptime, and its gate applies after every startup, so it does not need to detect a backend that died during the wait.

## Objections a reviewer is likely to raise

### 1. Session termination still exposes unreplicated commits without any restart

A session killer, or a pooler that drops server connections on timeout, terminates a backend stuck in the sync-rep wait while the standby is unreachable. The local commit becomes visible, a retry sees the row and succeeds, and the primary keeps running, so the post-recovery gate never fires. A later failover loses the row.

v3 keeps ProcDiePending honored on purpose (message #24), which follows the SIGTERM concern recorded in [P4]. The gate only covers the window after postmaster start. Message #14 says the needed LSN is the end-of-recovery LSN, but that holds only for restarts. Amit Kapila's system-wide LSN idea (messages #4, #10, #13) covers the no-restart case and has no answer against v3, so the claim that the two patches together close the hole needs to be narrowed or defended.

Precedent: [GUC to prevent cancellation while waiting for sync rep](https://www.postgresql.org/message-id/flat/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz) (Withdrawn, 2021-07-13)

Precedent: [GUC to disable cancellation of awaiting for synchronous replication](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru) (Returned with feedback, 2021-10-06)

**Question:** After pg_terminate_backend() ends a sync-rep wait on a primary that keeps running, what stops other sessions from reading that unreplicated commit?

<sub>fixable, level 2, WAL / replication / standby; messages [#4](https://www.postgresql.org/message-id/CAA4eK1L9fMPfqabjEBc5Ldro9rhKz7pMdQ4LMx1vzHOHS8W%3Dug%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CADzfLwWLUXXQ%3DrYg2L6ZREbBu%2BzsZSBXFk0rJoAPvXKV9%3D9%2B9Q%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAA4eK1JE0XDmn91H%3DAHrfPXghSJFK_NGY%2BP8HqCtvWN7nOwAsg%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/1591B71B-3BC7-4F1F-BFAF-69C51E144299%40yandex-team.ru), [#24](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru); `replication-ondisk-1`</sub>

### 2. Two permanent GUCs fixed while the GUC, hook or snapshot choice is open

HA tools write synchronous_replication_wait_on_query_cancel and post_recovery_sync_level into their configs once a release ships them. Later the project chooses a hook or LSN-based snapshot visibility for the same problem. Both GUCs must then be kept, documented and tested next to the new mechanism, or their removal breaks those configs at upgrade.

Message #24 asks whether this should be a GUC, a hook or both, and no reply in the thread answers it. Messages #2 and #7 propose making visibility depend on replication state, and message #4 proposes a system-wide flag. No reply weighs these against v3. A committer would likely want that design choice settled before the project commits to two GUC names in the commit path.

Precedent: [GUC to disable cancellation of awaiting for synchronous replication](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru) (Returned with feedback, 2021-10-06)

**Question:** If a hook or LSN-based snapshot visibility is added later, which of these two GUCs stays useful, and which becomes a compatibility burden?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CAEze2Wg_Deq0XaEBTwucFc2UuiWT55vCwXyrZMf4GXX_9H4RWw%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAA4eK1L9fMPfqabjEBc5Ldro9rhKz7pMdQ4LMx1vzHOHS8W%3Dug%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CANwKhkMjV%3D_BE3Ee2JNZeSMQRMwcVoSfNzv33U%3DFGwqzqG1UwA%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru); `maintenance-fit-1`</sub>

### 3. WAL sender exemption may let database-connected replication roles read gated data

A role with the REPLICATION attribute connects with replication=database while the gate is closed after a restart. It runs SQL in walsender mode, or a logical subscriber decodes changes, and sees data a later failover loses. The operator finds out only when the subscriber or reporting job has rows the new primary lacks.

The commit message for 0001 and the docs say "WAL sender connections are allowed while the check is pending" and do not separate physical from logical WAL senders. A logical WAL sender is bound to a database, can run ordinary SQL, and streams decoded commits to subscribers. If the exemption covers it, any role with REPLICATION and a matching pg_hba entry passes the gate, which defeats the stated goal that no client sees unreplicated data. Physical standbys are the only connections that need the exemption so that they can catch up.

**Question:** Does the exemption cover logical, database-connected WAL senders, and if so, why should they see or decode data the gate keeps from ordinary sessions?

<sub>fixable, level 2, security / privileges; messages [#24](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru), [#25](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com); `security-1`</sub>

### 4. pg_isready and PQping report a gated primary as healthy

An orchestrator restarts the primary while its standby is down, and its health check uses pg_isready. The check returns exit 0, so the tool routes traffic to the primary. Every application connection then fails with the new FATAL, and the operator learns this only from application errors.

Message #25 found that the gate raises its FATAL after authentication, and internal_ping() returns PQPING_OK once auth_req_received is set. The proposal targets HA tools, and those tools often use pg_isready or PQping as their readiness check. A client-visible state that existing tools read as "ready" is a contract problem. The thread has no answer yet on whether to fix this in libpq or in the docs.

**Question:** Should a gated primary report PQPING_REJECT (or similar) to pg_isready, and if libpq stays unchanged, how should HA tools detect the gated state?

<sub>fixable, level 2, design; messages [#25](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com); `standards-compat-1`</sub>

### 5. Startup gate can be built on the existing ClientAuthentication_hook

Core gains post_recovery_sync_level, a new FATAL path in InitPostgres and per-level shared state that every later change to startup and syncrep must keep correct. A preload module using ClientAuthentication_hook could reject non-walsender connections until the synchronous standbys pass the end-of-recovery LSN. HA vendors who want a different policy still need such a module.

ClientAuthentication_hook already runs for each new connection, and auth_delay shows a module can delay or reject sessions there. The module can record an LSN at or above end of recovery and compare it with the walsender state that syncrep keeps in shared memory. Message #24 argues for a GUC because it needs no preload library. That is convenience. The material does not show the job is impossible outside core.

**Question:** Which part of the post-recovery gate cannot be done in a preload module using ClientAuthentication_hook, and why must that part live in core?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#24](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru), [#25](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. After pg_terminate_backend() ends a sync-rep wait on a primary that keeps running, what stops other sessions from reading that non-replicated commit?
2. If a hook or LSN-based snapshot visibility is added later, which of the two GUCs stays useful and which becomes a compatibility burden?
3. Does the WAL sender exemption cover logical, database-connected WAL senders, and if so, why may they read or decode data the gate keeps from ordinary sessions?
4. Which state does the post-recovery gate need that a preload module using ClientAuthentication_hook, or an HA agent managing pg_hba.conf, cannot reach?
5. Is gating after a clean restart and after promotion intended, and how should pg_isready and PQping report a gated primary?

## Touch points

- **WAL / replication / standby:** Both patches change synchronous replication waits in syncrep.c and add end-of-recovery state in xlog.c.
- **wire protocol:** see objection standards-compat-1: pg_isready and PQping report a gated primary as healthy
- **ABI / extension API:** Message #24 asks whether to add a hook, and the patches change syncrep.h and walsender_private.h.
- **new GUC:** Adds post_recovery_sync_level and synchronous_replication_wait_on_query_cancel.
- **pg_dump / restore round-trip:** post_recovery_sync_level can be set per role or database, so dumps of role and database settings must carry it.
- **psql UX conventions:** see objection standards-compat-1: pg_isready and PQping report a gated primary as healthy
- **security / privileges:** WAL sender connections pass the gate, and the gate level is settable per role with SET privilege.
- **maintenance burden ("who maintains this in 10 years"):** Two permanent GUCs, a new FATAL path in InitPostgres and per-level shared state in syncrep.
- **performance on the hot path:** The gate adds a check to connection startup in postinit.c for every ordinary connection.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The gate may be possible with ClientAuthentication_hook or with pg_hba.conf managed by the HA agent.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows synchronous replication and connection startup. Amit Kapila and Fujii Masao commented earlier in the thread, but only a tester has replied to v3.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users of quorum synchronous replication with HA tools report real data loss, and three authors have tried this fix. The patches are small but touch syncrep.c, xlog.c and postinit.c, and the termination path and the GUC-or-hook choice are open.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
