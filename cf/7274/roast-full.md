# Design pre-review: Report changes discarded for relations not in the subscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7274](https://commitfest.postgresql.org/patch/7274/) (Needs review); authors: Hüseyin Demir (demir)
- Thread: [\[PATCH\] Report changes discarded for relations not in the subscription](https://www.postgresql.org/message-id/flat/CAB5wL7ZH%3DhOoJexa56S8Hh%2B1772DJT2P-hrieh5bohs5sZ3dBw%40mail.gmail.com), started 2026-09-08
- Messages read: 4 of 4
- Patch prose read: v2-0001-log-unapplied-changes-for-non-ready-relations.patch, v1-0001-report-unapplied-changes-for-non-ready-relations.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Nothing that survived stops the proposal at levels 1a, 1b or 2. The problem is stated with a field case, apply behavior does not change, and only subscribers that hit the condition see the new messages. The one surviving objection, standards-compat-1, is a fixable design point: the commit message names a missing conflict and statistics counter, and v2 adds neither. Nobody has commented on the design in the thread yet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, we could not check the cost of the new check per change, how the tracking survives relation map invalidation, or what parallel apply workers report.

## The proposal, restated

**Problem.** The apply worker silently discards changes for a relation that is not in SUBREL_STATE_READY, while the replication origin and the publisher slot advance past them. A table with no pg_subscription_rel row, such as one added to a publication without REFRESH PUBLICATION, loses those rows for good. Subscriber administrators cannot see this or tell it from an ongoing table sync.

**Proposed design.** should_apply_changes_for_rel() reports the skip, at LOG with a REFRESH PUBLICATION hint when the relation is not in the subscription, and at DEBUG2 when it is not yet synchronized. Each relation is reported once per sync state for the life of the apply worker, tracked in LogicalRepRelMapEntry. Apply behavior does not change, and v2 drops the v1 throttle based on wal_retrieve_retry_interval.

**User-visible surface.**
- New LOG message: skipping changes for relation "..." in subscription "...", with DETAIL that the relation is not part of the subscription and a HINT to run ALTER SUBSCRIPTION ... REFRESH PUBLICATION.
- New DEBUG2 message for changes skipped because the relation is not yet synchronized.
- New field in LogicalRepRelMapEntry (logicalrelation.h), visible to C code that uses this struct.

**What the author claims.**
- Today the discard leaves no log message at any level, no conflict and no statistics counter.
- The new messages let a user tell a lost table list apart from a missing REFRESH PUBLICATION.
- No behavior change beyond the new messages.
- Each relation is reported once per sync state, so the log does not get one message per row.
- The patch is for master only because it changes LogicalRepRelMapEntry.

**Thread state.** Hayato Kuroda asked for the upgrade versions and noted that pg_upgrade keeps pg_subscription_rel since 9a17be1e2 for PG17+; the author answered 14 to 18. No one has replied to v2, and no one has commented on the design.

## 2. Prior attempts

### Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4077/](https://commitfest.postgresql.org/patch/4077/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGPVpCQtF8LTM8svXdqMcMw8EmX4cNH=tGjwBW6ukJo2PEQcng@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGPVpCQtF8LTM8svXdqMcMw8EmX4cNH=tGjwBW6ukJo2PEQcng@mail.gmail.com)
- **Date:** decided 2022-12-27; started 2022-12-22
- **Outcome:** Withdrawn (cf)
- **Who objected:** Amit Kapila (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Amit Kapila questioned whether the current ERROR behavior should be changed when no specific use case had been given. He noted that users can avoid the failure loop with the disable_on_error subscription parameter, and that silently skipping changes could surprise users whose subscriber-side setup is incomplete. The author said he had no specific use case, and the entry was withdrawn.
- **Reason (quote):** "I see your point but not sure if it is worth changing the current behavior without any specific use case which we want to address with this change." ([message](https://www.postgresql.org/message-id/CAA4eK1KSEL%2Bb81L47MpdVCY79n0QgboxF6XTEjSc0ZcLkDzyWQ%40mail.gmail.com))
- **What is different this time:** P1 changed apply behavior, replacing an ERROR with a warning and a skip when a table was missing on the subscriber, and gave no use case. This proposal changes no behavior, only logs a skip that already happens silently, and gives a field case in message #1.


## 3. Objections a reviewer is likely to raise

### 1. Ad hoc LOG line where conflict logging and subscription stats exist

fixable · level 2, rubric item 12 · design, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A monitoring setup that tracks apply problems through pg_stat_subscription_stats sees no counter move while rows for an unrefreshed table are discarded. The single LOG line per worker lifetime is easy to miss in log rotation. The administrator finds the loss only when the subscriber table is compared with the publisher.

Postgres already reports apply-side anomalies through conflict types that are logged with a fixed format and counted in pg_stat_subscription_stats. The commit message names the missing counter and the missing conflict as part of the problem, but v2 adds neither. It adds a free-form message reported once per relation and sync state, so the log shows no ongoing count of discarded changes. A committer would likely ask why the existing reporting mechanism does not fit this case, or why it is deferred.

Thread: [#1 (Hüseyin Demir, 2026-09-08)](https://www.postgresql.org/message-id/CAB5wL7ZH%3DhOoJexa56S8Hh%2B1772DJT2P-hrieh5bohs5sZ3dBw%40mail.gmail.com), [#4 (Hüseyin Demir, 2026-10-30)](https://www.postgresql.org/message-id/CAB5wL7b74bpW3KhKxY-8_%2BXsyGRudirBR5-HuVAgjnR1kR53kw%40mail.gmail.com)

**Question:** Why is a once-per-worker LOG message preferred over a pg_stat_subscription_stats counter or a conflict type for permanently discarded changes?

Counter-check: held. The commit message names the missing conflict and statistics counter as part of the problem, and v2 adds neither. No message in the thread says why a once-per-worker LOG message is preferred over pg_stat_subscription_stats.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch adds reporting to the logical replication apply worker in worker.c; apply behavior stays the same. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a field to LogicalRepRelMapEntry in logicalrelation.h, which C code can see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author limits the patch to master because it changes LogicalRepRelMapEntry, so released branches keep the silent discard. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a user-visible LOG message with a hint, plus once-per-state tracking in the relation map entry. |
| performance on the hot path | **yes** | should_apply_changes_for_rel() runs once per applied change, and the new check sits in it. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is a LOG message reported once per apply worker lifetime preferred over a pg_stat_subscription_stats counter or a conflict type for permanently discarded changes?
2. Is the empty pg_subscription_rel after a 14-to-18 pg_upgrade expected, or is it a bug that needs a separate fix?
3. Is LOG the intended level for a message that repeats for each affected relation after every apply worker restart?
4. Does a parallel apply worker share the once-per-state tracking with the leader, or does each worker report the same relation again?
5. Will the logical replication chapter document that changes for a published table are discarded until REFRESH PUBLICATION runs?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the logical replication apply worker and table synchronization code would have to own it. None has spoken in the visible thread.
- **Liability:** Small: the wording and level of one LOG message that users may come to rely on, plus the once-per-state tracking in the relation map entry. A later counter or conflict type could make the message redundant.
- **Author follow-up:** The author answered the reviewer's question within a day and posted v2 (message #4).

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 11 KiB, touches worker.c, one header and two TAP tests, and makes a permanent silent discard visible to logical replication users. The open point in standards-compat-1 can be settled in normal review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` One LOG line per worker lifetime is a weak signal for permanent loss. Refuted (not specific): Duplicate of standards-compat-1. Both ask why the patch adds a one-time LOG message and no counter in pg_stat_subscription_stats.
- `already-exists-1` A catalog comparison already finds tables missing from the subscription. Refuted (no failure scenario): The scenario describes an alternative check and names no harm from the new message. That check needs access to both nodes and an administrator who already suspects a missed refresh, while the message reaches those who do not; the counter point duplicates standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). The one related precedent, [P1], was withdrawn because it changed apply behavior without a use case. This proposal changes no behavior and gives a use case in messages #1 and #4, so that reason does not apply. [P23] covers datatype mismatch warnings and records no reason, and the thread cites no earlier discussion of this idea.
- standards-compat: 1 objection(s). The patch changes no SQL syntax, GUC, catalog or dump format. Apply behaviour stays the same, so existing scripts and dumps are not affected.
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, catalog, wire protocol message or replicated state, so pg_upgrade, standbys and subscribers see nothing new. Its only cross-boundary effect is a new field in LogicalRepRelMapEntry; that struct lives only in apply worker memory, and the patch targets master only, so the ABI concern does not apply. Leader and parallel apply workers each keep their own tracking, so a skip may be logged more than once. That costs only extra log lines and loses no state.
- security: 0 objection(s). The proposal has no privilege or exposure aspect worth objecting to. It writes a qualified relation name and the subscription name to the server log, and apply worker errors already log these. The new messages are limited to one per relation per sync state for each worker lifetime and use the existing relation map entry, so a publisher-side user cannot flood the subscriber log or use unbounded memory beyond what the relation map already uses.
- maintenance-fit: 1 objection(s). From the maintenance angle the patch is small: one field in a backend-local struct, master only, and a cheap check per change. I see no hot-path, ABI or extension-fit problem worth a separate objection. No committer has replied, so ownership is open, but that is a committability point and not a flaw in the design.
- already-exists: 1 objection(s). One fixable objection only. No existing feature reports the discard from inside the apply worker. The existing way detects the cause, not the discarded changes.
- demand: 0 objection(s). The problem is real and stated. Adding a table to a publication without REFRESH PUBLICATION is an ordinary step, and today the apply worker drops its changes permanently and silently. The author also reports a production pg_upgrade incident. The cost is one LOG and one DEBUG2 message, and only subscribers that hit the condition pay it. [P1] does not apply: it proposed changing apply behavior, and this patch does not.

### Precedents retrieved

- P1: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - used in the review; reason summarised from its thread
- P2: [skip empty transactions for logical replication](https://commitfest.postgresql.org/patch/2384/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P3: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P4: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P5: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P6: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P8: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P9: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P10: [pg_upgrade: avoid failing if tablespace dirs exist and are empty](https://commitfest.postgresql.org/patch/2789/) (cf, Rejected, 2020-10-27) - judged not relevant
- P11: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P12: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant; reason summarised from its thread
- P13: [Logical Replication - revisit `is_table_publication` function implementation](https://commitfest.postgresql.org/patch/7042/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P14: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P15: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant
- P16: [pg_upgrade: Support for upgrading to checksums enabled](https://commitfest.postgresql.org/patch/5206/) (cf, Returned with feedback, 2025-04-03) - judged not relevant
- P17: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P18: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P19: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P20: [Revert "Validate subscription conninfo on owner change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ac230cd3ce93c4752489564cfe6b37b584a465ac) (revert, Reverted, 2026-08-05) - judged not relevant
- P21: [If pg_hba.conf has changed since last reload, emit a HINT to the client and server log](https://commitfest.postgresql.org/patch/50/) (cf, Returned with feedback, 2015-04-30) - judged not relevant
- P22: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P23: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - used in the review
- P24: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 117 s; cost reported by the CLI: $1.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 63 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:35:19.993Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
