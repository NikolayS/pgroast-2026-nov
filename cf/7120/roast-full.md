# Design pre-review: Small fixes needed by high-availability tools

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7120](https://commitfest.postgresql.org/patch/7120/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Small fixes needed by high-availability tools](https://www.postgresql.org/message-id/flat/0B44E464-BA62-4056-9465-3320DD2D0302%40yandex-team.ru), started 2025-05-02
- Messages read: 25 of 25
- Patch prose read: v3-0001-Prevent-early-data-access-after-crash-recovery.patch, v3-0002-Optionally-wait-for-synchronous-replication-after.patch, 0001-Allow-checking-standby-sync-before-making-data-visib.patch, 0002-Do-not-allow-to-cancel-locally-written-transaction.patch, 0003-Allow-reading-LSN-written-by-walreciever-but-not-flu.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Demand is shown and both settings are off by default, so level 1 does not stop the proposal. The design is still open at level 2. Session termination still releases a non-replicated commit on a running primary (replication-ondisk-1), and the author's own question of GUC, hook or both has no reply (maintenance-fit-1). The WAL sender exemption (security-1) and the reason the gate must live in core (already-exists-1) also need an answer before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 25 messages and no code were read. Whether v3 limits the WAL sender exemption to physical senders, and how the per-level shared state behaves on promotion, can only be checked in the code.

## The proposal, restated

**Problem.** After a crash or restart, a primary can show locally committed WAL that never reached the synchronous standbys, and a later failover loses it. A query cancel during a synchronous replication wait also makes non-replicated changes visible, so a retry appears to succeed. The target users are HA tools and applications built on quorum replication.

**Proposed design.** v3 adds two GUCs, both off by default. post_recovery_sync_level rejects ordinary connections after startup until the end-of-recovery WAL location reaches the synchronous standbys at the set level; WAL senders may connect, and an empty synchronous_standby_names bypasses the gate. synchronous_replication_wait_on_query_cancel makes the backend report but ignore query cancel during a sync-rep wait, while session termination and shutdown still end it.

**User-visible surface.**
- GUC post_recovery_sync_level (enum: remote_apply, on, remote_write, local, off), settable per role or database.
- GUC synchronous_replication_wait_on_query_cancel (boolean, postgresql.conf or command line only).
- New FATAL with ERRCODE_CANNOT_CONNECT_NOW for ordinary connections while the post-recovery check is pending.
- New log message "post-recovery synchronization wait skipped" when synchronous_standby_names is empty.
- Changed cancel behaviour in sync-rep wait: pg_cancel_backend() and statement_timeout no longer end the wait when the GUC is on.

**What the author claims.**
- Cancelling a sync-rep wait exposes non-replicated data, which retry patterns such as insert ... on conflict do nothing turn into data loss.
- The cancel fix is incomplete without the post-recovery connection gate.
- Both features keep existing behaviour by default.
- A GUC gives a simple cluster-wide policy without a preload library; a hook would need a narrow API because it runs after local commit with interrupts held.
- The problem has been discussed for almost seven years; a reply says cloud providers have applied a similar patch for over four years.

**Thread state.** The written-LSN function moved to its own thread and was dropped in v3 after Fujii Masao showed unflushed write positions are unreliable. João Rito tested v3 and raised four issues, and the author's question of GUC, hook or both has no answer yet.

## 2. Prior attempts

### GUC to disable cancellation of awaiting for synchronous replication (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2402/](https://commitfest.postgresql.org/patch/2402/)
- **Thread:** [https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru)
- **Date:** decided 2021-10-06; started 2019-12-20
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Fujii Masao (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers said that blocking cancel does not close the hole, because a server restart or a backend crash also exposes locally committed data that no standby has received. Fujii Masao suggested making new snapshots wait for replication instead, and Andrey Borodin rejected that because of its latency cost. Aleksander Alekseev called the patch a workaround and asked for a community decision, and the visible messages show no agreement before the entry was returned.
- **Reason (quote):** "I don't think that preventing a backend from being canceled during waiting for sync rep actually addresses your issue." ([message](https://www.postgresql.org/message-id/4f8d54c9-6f18-23d5-c4de-9d6656d3a408%40oss.nttdata.com))
- **What is different this time:** v3 pairs the cancel GUC with a post-recovery connection gate, which covers the restart and crash case that reviewers raised there.

### GUC to prevent cancellation while waiting for sync rep (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3212/](https://commitfest.postgresql.org/patch/3212/)
- **Thread:** [https://www.postgresql.org/message-id/flat/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz](https://www.postgresql.org/message-id/flat/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz)
- **Date:** decided 2021-07-13; started 2021-04-19
- **Outcome:** Withdrawn (cf)
- **Who objected:** Andrey Borodin (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew it. Andrey Borodin objected to ignoring termination, because HA tools stop the instance with SIGTERM and the alternatives, kill -9 or SIGABRT, give slower recovery or core dumps; he asked for separate GUCs for cancel and termination. The author judged that ignoring SIGTERM was not a good solution and had little support, and withdrew because a second patch for the same problem existed.
- **Reason (quote):** "OK. I don't think ignoring SIGTERM in the way my patch does it is a great solution, and it's not getting much support, so I think I'll back away from that idea." ([message](https://www.postgresql.org/message-id/c162f1d08a04c78c5127b37fcfc02527900a8029.camel%40j-davis.com))
- **What is different this time:** v3 ignores only query cancel and still honors session termination and shutdown, so it avoids the SIGTERM objection that led to that withdrawal.

### An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3665/](https://commitfest.postgresql.org/patch/3665/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALj2ACUrOB59QaE6=jF2cFAyv1MR7fzD8tr4YM5+OwEYG1SNzA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACUrOB59QaE6=jF2cFAyv1MR7fzD8tr4YM5+OwEYG1SNzA@mail.gmail.com)
- **Date:** decided 2023-08-02; started 2022-04-25
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Daniel Gustafsson, Laurenz Albe (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The entry was returned with feedback because the patch received no update for six months and no longer applied. The design was still open at the last author message. The author could not reliably detect, at restart, that a backend died while waiting for the sync replication ack, and the alternative of always waiting could keep the primary down. Earlier, Laurenz Albe objected that the approach still left some cases of transactions committed locally but not on the standby.
- **Reason (quote):** "Since there hasn't been any updates for the past six months, and the patch hasn't applied for a few months, I am marking this returned with feedback for now." ([message](https://www.postgresql.org/message-id/D52A9851-E4C5-40CD-98E5-F6BC0417ED0F%40yesql.se))
- **What is different this time:** v3 blocks cancel for the whole wait instead of a naptime, and its gate applies after every startup, so it does not need to detect a backend that died during the wait.


## 3. Objections a reviewer is likely to raise

### 1. Session termination still exposes unreplicated commits without any restart

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

A session killer, or a pooler that drops server connections on timeout, terminates a backend stuck in the sync-rep wait while the standby is unreachable. The local commit becomes visible, a retry sees the row and succeeds, and the primary keeps running, so the post-recovery gate never fires. A later failover loses the row.

v3 keeps ProcDiePending honored on purpose (message #24), which follows the SIGTERM concern recorded in [P4]. The gate only covers the window after postmaster start. Message #14 says the needed LSN is the end-of-recovery LSN, but that holds only for restarts. Amit Kapila's system-wide LSN idea (messages #4, #10, #13) covers the no-restart case and has no answer against v3, so the claim that the two patches together close the hole needs to be narrowed or defended.

**Precedents.**
- [GUC to prevent cancellation while waiting for sync rep](https://commitfest.postgresql.org/patch/3212/) (Withdrawn, 2021-07-13); thread: [link](https://www.postgresql.org/message-id/flat/cac4b9df-92c6-77aa-687b-18b86cb13728@stratox.cz)
- [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (Returned with feedback, 2021-10-06); thread: [link](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru)

Thread: [#4 (Amit Kapila, 2025-05-12)](https://www.postgresql.org/message-id/CAA4eK1L9fMPfqabjEBc5Ldro9rhKz7pMdQ4LMx1vzHOHS8W%3Dug%40mail.gmail.com), [#10 (Mihail Nikalayeu, 2025-05-13)](https://www.postgresql.org/message-id/CADzfLwWLUXXQ%3DrYg2L6ZREbBu%2BzsZSBXFk0rJoAPvXKV9%3D9%2B9Q%40mail.gmail.com), [#13 (Amit Kapila, 2025-05-14)](https://www.postgresql.org/message-id/CAA4eK1JE0XDmn91H%3DAHrfPXghSJFK_NGY%2BP8HqCtvWN7nOwAsg%40mail.gmail.com), [#14 (Andrey Borodin, 2025-05-14)](https://www.postgresql.org/message-id/1591B71B-3BC7-4F1F-BFAF-69C51E144299%40yandex-team.ru), [#24 (Andrey Borodin, 2026-07-30)](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru)

**Question:** After pg_terminate_backend() ends a sync-rep wait on a primary that keeps running, what stops other sessions from reading that unreplicated commit?

Counter-check: held. Message #24 keeps ProcDiePending unchanged, and the gate acts only after a postmaster start. No message in the thread says what protects a commit released by pg_terminate_backend() on a primary that keeps running.

### 2. Two permanent GUCs fixed while the GUC, hook or snapshot choice is open

fixable · level 2, rubric item 6 · maintenance burden ("who maintains this in 10 years"), new GUC, ABI / extension API · `maintenance-fit-1`

HA tools write synchronous_replication_wait_on_query_cancel and post_recovery_sync_level into their configs once a release ships them. Later the project chooses a hook or LSN-based snapshot visibility for the same problem. Both GUCs must then be kept, documented and tested next to the new mechanism, or their removal breaks those configs at upgrade.

Message #24 asks whether this should be a GUC, a hook or both, and no reply in the thread answers it. Messages #2 and #7 propose making visibility depend on replication state, and message #4 proposes a system-wide flag. No reply weighs these against v3. A committer would likely want that design choice settled before the project commits to two GUC names in the commit path.

**Precedents.**
- [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (Returned with feedback, 2021-10-06); thread: [link](https://www.postgresql.org/message-id/flat/C1F7905E-5DB2-497D-ABCC-E14D4DEE506C@yandex-team.ru)

Thread: [#2 (Matthias van de Meent, 2025-05-06)](https://www.postgresql.org/message-id/CAEze2Wg_Deq0XaEBTwucFc2UuiWT55vCwXyrZMf4GXX_9H4RWw%40mail.gmail.com), [#4 (Amit Kapila, 2025-05-12)](https://www.postgresql.org/message-id/CAA4eK1L9fMPfqabjEBc5Ldro9rhKz7pMdQ4LMx1vzHOHS8W%3Dug%40mail.gmail.com), [#7 (Ants Aasma, 2025-05-12)](https://www.postgresql.org/message-id/CANwKhkMjV%3D_BE3Ee2JNZeSMQRMwcVoSfNzv33U%3DFGwqzqG1UwA%40mail.gmail.com), [#24 (Andrey Borodin, 2026-07-30)](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru)

**Question:** If a hook or LSN-based snapshot visibility is added later, which of these two GUCs stays useful, and which becomes a compatibility burden?

Counter-check: held. Messages #5, #14 and #15 do answer the CSN and flag ideas, so that part is overstated, and [P3] must not be used as support. The GUC-or-hook question in message #24 is the author's own and has no reply yet.

### 3. WAL sender exemption may let database-connected replication roles read gated data

fixable · level 2, rubric item 7 · security / privileges, WAL / replication / standby · `security-1`

A role with the REPLICATION attribute connects with replication=database while the gate is closed after a restart. It runs SQL in walsender mode, or a logical subscriber decodes changes, and sees data a later failover loses. The operator finds out only when the subscriber or reporting job has rows the new primary lacks.

The commit message for 0001 and the docs say "WAL sender connections are allowed while the check is pending" and do not separate physical from logical WAL senders. A logical WAL sender is bound to a database, can run ordinary SQL, and streams decoded commits to subscribers. If the exemption covers it, any role with REPLICATION and a matching pg_hba entry passes the gate, which defeats the stated goal that no client sees unreplicated data. Physical standbys are the only connections that need the exemption so that they can catch up.

Thread: [#24 (Andrey Borodin, 2026-07-30)](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru), [#25 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com)

**Question:** Does the exemption cover logical, database-connected WAL senders, and if so, why should they see or decode data the gate keeps from ordinary sessions?

Counter-check: held. The docs and commit message exempt "WAL sender connections" without separating physical from database-connected ones, and a replication=database connection can run SQL. Only the code can show whether v3 limits the exemption, and the prose does not say so.

### 4. pg_isready and PQping report a gated primary as healthy

fixable · level 2, rubric item 13 · design, wire protocol, psql UX conventions · `standards-compat-1`

An orchestrator restarts the primary while its standby is down, and its health check uses pg_isready. The check returns exit 0, so the tool routes traffic to the primary. Every application connection then fails with the new FATAL, and the operator learns this only from application errors.

Message #25 found that the gate raises its FATAL after authentication, and internal_ping() returns PQPING_OK once auth_req_received is set. The proposal targets HA tools, and those tools often use pg_isready or PQping as their readiness check. A client-visible state that existing tools read as "ready" is a contract problem. The thread has no answer yet on whether to fix this in libpq or in the docs.

Thread: [#25 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com)

**Question:** Should a gated primary report PQPING_REJECT (or similar) to pg_isready, and if libpq stays unchanged, how should HA tools detect the gated state?

Counter-check: held. Message #25 reproduced pg_isready returning exit 0 while every ordinary connection got the new FATAL. The thread has no reply yet, and HA tools are the users this proposal names.

### 5. Startup gate can be built on the existing ClientAuthentication_hook

fixable · level 1b, rubric item 2 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

Core gains post_recovery_sync_level, a new FATAL path in InitPostgres and per-level shared state that every later change to startup and syncrep must keep correct. A preload module using ClientAuthentication_hook could reject non-walsender connections until the synchronous standbys pass the end-of-recovery LSN. HA vendors who want a different policy still need such a module.

ClientAuthentication_hook already runs for each new connection, and auth_delay shows a module can delay or reject sessions there. The module can record an LSN at or above end of recovery and compare it with the walsender state that syncrep keeps in shared memory. Message #24 argues for a GUC because it needs no preload library. That is convenience. The material does not show the job is impossible outside core.

Thread: [#24 (Andrey Borodin, 2026-07-30)](https://www.postgresql.org/message-id/7FBC2251-A9A5-4B1C-944B-C575DE984C18%40yandex-team.ru), [#25 (João Rito, 2026-11-11)](https://www.postgresql.org/message-id/CADMs5OMjJY-oOQ%3D5qR_2WRyvzspAp3HuCx%3DyLKKtJ_OJFDYuvQ%40mail.gmail.com)

**Question:** Which part of the post-recovery gate cannot be done in a preload module using ClientAuthentication_hook, and why must that part live in core?

Counter-check: held. Message #24 gives a reason against a hook only for the cancel path, which runs after local commit with interrupts held. The thread does not say which state the connection gate needs that a preload module using ClientAuthentication_hook cannot read.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Both patches change synchronous replication waits in syncrep.c and add end-of-recovery state in xlog.c. |
| wire protocol | **yes** | see objection standards-compat-1: pg_isready and PQping report a gated primary as healthy |
| catalog changes | no |  |
| ABI / extension API | **yes** | Message #24 asks whether to add a hook, and the patches change syncrep.h and walsender_private.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds post_recovery_sync_level and synchronous_replication_wait_on_query_cancel. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | post_recovery_sync_level can be set per role or database, so dumps of role and database settings must carry it. |
| psql UX conventions | **yes** | see objection standards-compat-1: pg_isready and PQping report a gated primary as healthy |
| security / privileges | **yes** | WAL sender connections pass the gate, and the gate level is settable per role with SET privilege. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two permanent GUCs, a new FATAL path in InitPostgres and per-level shared state in syncrep. |
| performance on the hot path | **yes** | The gate adds a check to connection startup in postinit.c for every ordinary connection. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The gate may be possible with ClientAuthentication_hook or with pg_hba.conf managed by the HA agent. |

## 5. Questions the author must answer before anyone reviews code

1. After pg_terminate_backend() ends a sync-rep wait on a primary that keeps running, what stops other sessions from reading that non-replicated commit?
2. If a hook or LSN-based snapshot visibility is added later, which of the two GUCs stays useful and which becomes a compatibility burden?
3. Does the WAL sender exemption cover logical, database-connected WAL senders, and if so, why may they read or decode data the gate keeps from ordinary sessions?
4. Which state does the post-recovery gate need that a preload module using ClientAuthentication_hook, or an HA agent managing pg_hba.conf, cannot reach?
5. Is gating after a clean restart and after promotion intended, and how should pg_isready and PQping report a gated primary?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows synchronous replication and connection startup. Amit Kapila and Fujii Masao commented earlier in the thread, but only a tester has replied to v3.
- **Liability:** Two GUC names that HA tools will write into configs, a new connection rejection path, and a durability claim that users will rely on during failover, for as long as the GUCs exist.
- **Author follow-up:** The author has pursued this problem since 2019 ([P3]), posted three versions here and moved the entry to the next CommitFest.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users of quorum synchronous replication with HA tools report real data loss, and three authors have tried this fix. The patches are small but touch syncrep.c, xlog.c and postinit.c, and the termination path and the GUC-or-hook choice are open.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Cancel GUC silently disables statement_timeout and driver query timeouts. Not listed: not among the strongest objections.
- `standards-compat-3` (fixable, level 2) Gate applies on every startup and promotion, not only after crash recovery. Not listed: not among the strongest objections.
- `already-exists-2` (fixable, level 1b) HA tools can already gate startup with pg_hba.conf and restart_after_crash. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` Session termination still exposes non-replicated commits, the gap that stopped earlier attempts. Refuted (not specific): Duplicate of replication-ondisk-1, which states the no-restart case more exactly. The gap recorded in [P3] was restart and backend crash, which the v3 gate covers, so [P3] does not support the termination point.
- `replication-ondisk-2` Logical WAL senders pass the gate and can stream unreplicated commits. Refuted (not specific): Duplicate of security-1. The streaming part also exists today without the gate: logical walsenders do not wait for synchronous standbys unless synchronized_standby_slots is set.
- `replication-ondisk-3` Empty synchronous_standby_names bypass opens the gate when HA tools clear it. Refuted (no failure scenario): Emptying synchronous_standby_names already releases every sync-rep wait today, so a tool that clears it has chosen availability over durability for all commits. The gate follows that existing rule, and with an empty list there are no synchronous standbys to wait for.
- `maintenance-fit-2` Connection gate does not say why ClientAuthentication_hook is not enough. Refuted (not specific): Duplicate of already-exists-1, which names the hook, the auth_delay example and the shared state a module would read.
- `maintenance-fit-3` Gated primary looks healthy to pg_isready and PQping. Refuted (not specific): Duplicate of standards-compat-1.

### Review angles

- prior-attempts: 1 objection(s). The restart objection from the earlier attempts is answered by the v3 gate. Only the termination path remains open from the prior-attempt reasons.
- standards-compat: 3 objection(s). Objections are fixable contract issues. The pg_dump round-trip of per-role settings and the startup-option bypass path depend on code that is not shown.
- replication-ondisk: 3 objection(s). There is no on-disk, catalog, WAL record or wire protocol change. Post-recovery completion state lives only in shared memory until the next postmaster start.
- security: 1 objection(s). Apart from the WAL sender exemption, both settings are superuser-controlled, off by default and need no new privilege. Ignoring query cancel and statement_timeout hurts availability, but pg_terminate_backend still ends the wait and no user gains new rights.
- maintenance-fit: 3 objection(s). Committability: no committer has reviewed v3 in the thread. Fujii Masao commented only on the dropped write-LSN patch, and only a tester has replied since message #24. This should not raise the verdict by itself.
- already-exists: 2 objection(s). No objection from this angle on the cancel GUC: no existing feature or hook stops a query cancel from ending SyncRepWaitForLSN, so it cannot be done outside core today.
- demand: 0 objection(s). The material shows real, lasting demand from several independent sources. Three authors proposed the same fix over about seven years ([P3], [P4], [P1]); message #10 says most cloud providers have applied a similar patch for over four years, and message #7 cites a Jepsen report. HA tool authors in messages #11 and #20 and the tester in message #25 describe concrete failover data loss, and both GUCs are off by default, so the cost falls on users who enable them.

### Precedents retrieved

- P1: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - used in the review; reason summarised from its thread
- P2: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P3: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - used in the review; reason summarised from its thread
- P4: [GUC to prevent cancellation while waiting for sync rep](https://commitfest.postgresql.org/patch/3212/) (cf, Withdrawn, 2021-07-13) - used in the review; reason summarised from its thread
- P5: [Quorum commit for multiple synchronous replication](https://commitfest.postgresql.org/patch/696/) (cf, Returned with feedback, 2016-10-02) - judged not relevant; reason summarised from its thread
- P6: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P7: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P8: [Automatic notification for top transaction IDs](https://commitfest.postgresql.org/patch/3198/) (cf, Withdrawn, 2021-07-01) - used in the review; reason summarised from its thread
- P9: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P10: ["synchronous_replay" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/951/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P11: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant
- P12: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P13: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P14: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P15: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [Issue in postgres_fdw causing unnecessary wait for cancel request reply](https://commitfest.postgresql.org/patch/4329/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P17: [SimpleLruTruncate() mutual exclusion (data loss from lack thereof)](https://commitfest.postgresql.org/patch/2025/) (cf, Withdrawn, 2020-01-25) - judged not relevant
- P18: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P19: [Add last commit LSN to pg_last_committed_xact()](https://commitfest.postgresql.org/patch/3515/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P20: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Support multiple synchronous standby servers](https://commitfest.postgresql.org/patch/293/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P22: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P23: [delay starting WAL receiver](https://commitfest.postgresql.org/patch/4117/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P24: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 196 s; cost reported by the CLI: $3.20
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 122 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:21:29.279Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
