# Design pre-review: Archive-fed logical decoding: pausing recovery on slot conflict

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7121](https://commitfest.postgresql.org/patch/7121/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Archive-fed logical decoding: pausing recovery on slot conflict](https://www.postgresql.org/message-id/flat/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), started 2026-06-05
- Messages read: 3 of 3
- Patch prose read: v2-0001-Pause-recovery-before-invalidating-logical-slots.patch, v2-0002-Test-recovery-pause-on-logical-slot-conflict.patch, v1-0001-xlogrecovery-make-ConfirmRecoveryPaused-and-Check.patch, v1-0003-Auto-resume-recovery-once-the-logical-slot-confli.patch, v1-0002-Add-recovery_pause_on_logical_slot_conflict-GUC.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is plausible and a reviewer supports it, but the design is not settled. v2 admits that some pauses cannot resolve (replication-ondisk-1) and nothing bounds them (standards-compat-1). The per-slot question from message #1 is still open (standards-compat-3), and the proposal does not say how the wait inside redo handles reload and shutdown (prior-attempts-1). These are fixable, but the list needs to agree on them before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check how the wait loop handles interrupts, or whether the pause and auto-resume are free of races with slot invalidation and concurrent decoding. It also could not check the claim of one boolean test with the GUC off.

## The proposal, restated

**Problem.** A standby fed only by restore_command has no walreceiver, so it cannot send hot_standby_feedback to the primary. Catalog prune and B-tree delete records then conflict with the local logical slot's catalog_xmin, and the slot is invalidated, roughly 2 * autovacuum_naptime after creation. This blocks continuous CDC from a WAL archive without load on the primary.

**Proposed design.** A new GUC, recovery_pause_on_logical_slot_conflict, makes ResolveRecoveryConflictWithSnapshot() pause recovery before it would invalidate a local logical slot. Recovery resumes automatically once normal logical decoding has advanced the slot's effective xmin horizons past the conflict, using the normal invalidation predicate. In v2 the startup process no longer changes slot horizons; pg_wal_replay_resume() lets the slot be invalidated and promotion ends the pause.

**User-visible surface.**
- New GUC recovery_pause_on_logical_slot_conflict (boolean, PGC_SIGHUP, default off).
- pg_wal_replay_resume() during a conflict pause now means "give up and invalidate the slot".
- New recovery pause cause, visible through pg_get_wal_replay_pause_state() and one log line at pause start.
- Synchronized slots are excluded from the protection.

**What the author claims.**
- The catalog conflict is the only fundamental blocker for archive-fed logical decoding; wal_level and WAL removal are already operator knobs.
- With the GUC off, the cost is one boolean early return.
- The feature reuses the existing recovery pause machinery and adds no shared memory.
- A TAP test and a WAL-G field test with pg_recvlogical show the slot survives with no gaps, while a GUC-off control standby loses its slot.
- Consumers such as pg_recvlogical and Debezium should see ordinary standby decoding with occasional stalls.

**Thread state.** Andrey posted v1, then v2 after self-review found v1's startup-process advance of catalog_xmin unsafe. Ajin Cherian likes the idea but asks for a maximum pause duration and repeated warnings, because a dead consumer would stall the standby forever.

## 2. Prior attempts

### Use latch instead of pg_usleep for standby recovery conflicts (related)

- **Link:** [https://commitfest.postgresql.org/patch/977/](https://commitfest.postgresql.org/patch/977/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANP8+j+WWawZCR+w=887FcnqQibFf=j=kag98XNN9rAOM-4HTg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANP8+j+WWawZCR+w=887FcnqQibFf=j=kag98XNN9rAOM-4HTg@mail.gmail.com)
- **Date:** decided 2017-04-03; started 2017-01-26
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, "Tsunakawa, Takayuki" (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Review found that CHECK_FOR_INTERRUPTS() does nothing in the startup process and that WL_LATCH_SET has no effect because MyLatch is never set there. Michael Paquier then judged that calling HandleStartupProcInterrupts() would reload the GUC context in the middle of replaying a record. He concluded that all conflict code paths should use a latch, and returned the patch with feedback as time ran short.
- **Reason (quote):** "Calling HandleStartupProcInterrupts() is actually dangerous I think, because it would reload the GUC context while replaying a record." ([message](https://www.postgresql.org/message-id/CAB7nPqSMu1BAPgHK2aqmYm%3D7f__VzwvXDCVHqYLZzbH1HFbE-Q%40mail.gmail.com))
- **What is different this time:** P12 changed how an existing conflict wait sleeps. This proposal adds a new wait at the same place inside record replay, so P12's concern about reloading configuration in the middle of a record applies to it.


## 3. Objections a reviewer is likely to raise

### 1. Some pauses cannot resolve because of WAL ordering, and nothing bounds them

fixable · level 2, rubric item 7 · WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

On the primary, a catalog row deleted by xid X is pruned on access soon after X commits, before the next xl_running_xacts is logged. The standby pauses before the prune record. Decoding can read only up to the replay position, so it never sees a running_xacts newer than X, and the standby waits until someone notices.

Message #2 and the docs say a pause may not be resolvable, but the patch does not say how often this happens or why. Decoding gets a new catalog_xmin candidate from xl_running_xacts records. Any running_xacts that would move the horizon past the prune horizon may sit after the pause LSN, where decoding cannot reach it. That would make the stall a regular event under catalog churn, and the auto-resume promise that message #1 makes depends on how often it happens.

Thread: [#1 (Andrey Borodin, 2026-06-05)](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#2 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru), [#3 (Ajin Cherian, 2026-09-16)](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com)

**Question:** Under realistic catalog churn, what fraction of conflict pauses can decoding resolve from WAL already replayed, and what ends the others without an operator?

Counter-check: held. Decoding advances catalog_xmin only from xl_running_xacts records it has read, and a prune record can precede the first such record that passes its horizon. Message #2 admits unresolvable pauses but gives no frequency, and nothing ends them without an operator.

### 2. Unbounded conflict wait departs from the max_standby_*_delay pattern

fixable · level 2, rubric item 12 · new GUC, WAL / replication / standby · `standards-compat-1`

The pg_recvlogical process on an archive-fed standby crashes overnight. Recovery pauses at the next catalog prune record and stays paused, while cascading standbys and readers see data that keeps getting older. Monitoring that checks only for invalidated slots or walreceiver state reports nothing wrong.

Every existing recovery conflict wait in Postgres is bounded by max_standby_archive_delay or max_standby_streaming_delay, and waiting forever requires the explicit value -1. The proposal adds a conflict wait with no bound, controlled by an on/off boolean. Message #3 asks for a maximum pause duration and repeated warnings, and v2 gives no answer. If a delay setting is added later, the boolean stays behind as a second knob that cannot be removed.

Thread: [#2 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru), [#3 (Ajin Cherian, 2026-09-16)](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com)

**Question:** Why should this conflict wait be an unbounded boolean, when other recovery conflict waits use a delay setting where -1 means wait forever?

Counter-check: held. Existing recovery conflict waits take their limit from max_standby_archive_delay or max_standby_streaming_delay, and the proposal adds a boolean with no limit. Message #3 asks for a ceiling and the visible thread has no reply.

### 3. Cluster-wide GUC for behaviour that slot properties already express

fixable · level 2, rubric item 12 · new GUC, WAL / replication / standby · `standards-compat-3`

A standby serves as a failover target and also hosts a CDC slot and a test slot that someone left behind. With the GUC on, the abandoned test slot stalls replay for the whole standby. With the GUC off, the CDC slot is invalidated. No setting protects only the CDC slot.

Per-slot behaviour such as failover and two_phase is set as a slot property at creation. Here a server-wide GUC decides how every local logical slot reacts to conflicts. Message #1 asks the same question (item 2) and calls the GUC coarse, and v2 keeps the GUC without saying why. If a per-slot option is added later, the GUC becomes a lasting second interface.

Thread: [#1 (Andrey Borodin, 2026-06-05)](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#2 (Andrey Borodin, 2026-09-05)](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru)

**Question:** Why is a server-wide GUC preferable to a per-slot option, given that the thread itself raised the per-slot option and existing slot behaviours are set per slot?

Counter-check: held. Message #1 calls the cluster-wide GUC coarse and asks about a per-slot option, and v2 keeps the GUC without giving a reason. The abandoned test slot case is concrete, and no setting in the proposal protects only one slot.

### 4. Unbounded wait inside record replay versus startup interrupt handling

fixable · level 2, rubric item 7 · WAL / replication / standby, new GUC · `prior-attempts-1`

A consumer dies and recovery pauses inside ResolveRecoveryConflictWithSnapshot() for hours. The operator sets recovery_pause_on_logical_slot_conflict = off and reloads, or runs pg_ctl stop. Either the startup process never handles the SIGHUP or shutdown request, or it handles it while a record is half replayed.

Existing recovery pauses happen between records in the main loop, and other conflict waits inside redo are bounded by max_standby_*_delay. This pause sits inside a record's redo and has no bound (message #3). In [P12], the recorded reason was that calling HandleStartupProcInterrupts() there would reload the GUC context while a record is being replayed. v1-0001 lists only ConfirmRecoveryPaused() and CheckForStandbyTrigger() for the wait loop, and does not say how config reload and shutdown are handled.

**Precedents.**
- [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (Returned with feedback, 2017-04-03); thread: [link](https://www.postgresql.org/message-id/flat/CANP8+j+WWawZCR+w=887FcnqQibFf=j=kag98XNN9rAOM-4HTg@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-06-05)](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#3 (Ajin Cherian, 2026-09-16)](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com)

**Question:** How does the conflict wait handle SIGHUP and shutdown requests during a long pause, and why is that safe in the middle of replaying a record?

Counter-check: held. The patch prose names only ConfirmRecoveryPaused() and CheckForStandbyTrigger() for the wait loop, and does not say how reload or shutdown is handled. One correction: existing conflict waits are also unbounded when max_standby_*_delay is -1, so [P12] supports the reload concern only.

### 5. Primary-side placeholder slot can already hold catalog_xmin without streaming

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), WAL / replication / standby, new GUC · `already-exists-1`

An operator who can run one SQL connection to the primary keeps a logical slot there. A small job advances it to the standby slot's restart_lsn, so the primary never prunes catalog rows the standby still needs. With this setup the core GUC adds nothing, and the stalls it causes on the archive-fed standby can be avoided.

Message #1, question 1, names an out-of-band channel that publishes the standby's catalog_xmin back to the primary as a possible long-term answer. Postgres can already build that channel with pg_create_logical_replication_slot and pg_replication_slot_advance on the primary. The author's goal is no load on, and no dependency on, the primary. That makes the existing way less convenient, because it keeps WAL on the primary and needs a connection, but it does not make it impossible. A committer would likely ask which users truly cannot open any connection to the primary.

Thread: [#1 (Andrey Borodin, 2026-06-05)](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#3 (Ajin Cherian, 2026-09-16)](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com)

**Question:** Which deployments cannot run a primary-side logical slot advanced by an external job, and why is a recovery pause on the standby better for them?

Counter-check: held. A logical slot on the primary, advanced by an external job, can hold catalog_xmin back without a streaming standby. The proposal states the goal of no load on the primary but does not show which deployments cannot open a connection to it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch adds a wait inside ResolveRecoveryConflictWithSnapshot() that pauses replay before a logical slot is invalidated. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds recovery_pause_on_logical_slot_conflict, a PGC_SIGHUP boolean that is off by default. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | With the GUC on, any role that owns an idle local logical slot can hold back replay for the whole standby. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second pause point inside record replay and a new pause cause must be kept working through later changes to recovery. |
| performance on the hot path | **yes** | The check runs on the replay path for snapshot conflicts; message #1 says the cost with the GUC off is one boolean test. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #1 asks whether an out-of-band feedback channel or a separate decoding-from-archive tool is the better layer. |

## 5. Questions the author must answer before anyone reviews code

1. Under realistic catalog churn, what fraction of conflict pauses can decoding resolve from WAL already replayed, and what ends the others without an operator?
2. Will the pause get a time limit as message #3 asks, and if so, why not a delay setting where -1 means wait forever, like max_standby_archive_delay?
3. Why is a server-wide GUC preferable to a per-slot option, given that an abandoned slot can stall replay for the slot the operator wants to protect?
4. How does the conflict wait handle SIGHUP and shutdown requests during a long pause, and why is that safe in the middle of replaying a record?
5. Which deployments cannot keep a logical slot on the primary advanced by an external job, and why is a recovery pause on the standby better for them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the startup process, recovery conflict handling and logical slot invalidation on standbys. The visible thread shows none taking part in that role.
- **Liability:** The owner signs for a new GUC, a new pause cause and a second wait point inside redo. All three must stay correct through later changes to recovery pause and slot invalidation, for as long as the GUC exists.
- **Author follow-up:** The author posted v2 after a self-review that found a safety problem in v1. The visible thread has no reply to message #3, and the entry has needed a rebase since 2026-09-25.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Message #1 limits the use to dedicated decoding standbys fed by restore_command, and shows demand only from the authors' own field test. The patch is small but changes the startup process and slot invalidation, and the pause bound and per-slot questions are open.

## Appendix

### Held up in the counter-check, not listed above

- `demand-1` (fixable, level 1b) Demand rests on one team's use case, with no outside users shown. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` pg_wal_replay_resume() gains a destructive meaning existing tools cannot see. Refuted (no failure scenario): The documentation hunk says an explicit resume keeps the default invalidation, which is what happens today without the GUC. A script that resumes every pause already ends recovery at a recovery_target pause, so pg_wal_replay_resume() is not free of lasting effects today.
- `replication-ondisk-2` Automatic pause also stalls cascading standbys and primaries using remote_apply. Refuted (not specific): Duplicate of standards-compat-1, which asks for the same time limit. The remote_apply and cascading stalls already occur when an operator sets max_standby_streaming_delay = -1, and here too the cost follows an explicit setting.
- `security-1` Any REPLICATION user can halt standby replay with an idle slot. Refuted (not specific): Duplicate of standards-compat-3, which covers the per-slot opt-in. The privilege angle adds little, because with max_standby_streaming_delay = -1 any role that can run a query can already stall replay.
- `maintenance-fit-1` Cluster-wide GUC stalls all replay for one slot's benefit. Refuted (not specific): Duplicate of standards-compat-3, which makes the same per-slot argument with a more concrete scenario.
- `maintenance-fit-2` Unbounded pause will likely need more knobs to be safe. Refuted (not specific): Duplicate of standards-compat-1, which already says that a later delay setting would leave the boolean behind as a second knob.
- `maintenance-fit-3` New wait inside record replay needs a stated rule for interrupts. Refuted (not specific): Duplicate of prior-attempts-1, which covers both the reload and the shutdown case for the same wait loop.

### Review angles

- prior-attempts: 1 objection(s). The logical-decoding-on-standby entries ([P2], [P3], [P11], [P13]) built the base feature this proposal extends. They died of inactivity and failing tests, and those reasons do not apply here, so they are not listed.
- standards-compat: 3 objection(s). No pg_dump, catalog or SQL-syntax concerns: the proposal adds only a GUC and changes recovery-pause semantics.
- replication-ondisk: 2 objection(s). v2 no longer writes slot horizons or flushes slots from the startup process, so the crash-safety concern from v1 no longer applies. There is no new WAL record, catalog or on-disk format.
- security: 1 objection(s). The feature adds no new SQL-visible data, no change to authentication and no new path for running user code as a privileged role. The only exposure is the replay stall that slot owners can now cause.
- maintenance-fit: 3 objection(s). Objections were found from the maintenance and fit angle.
- already-exists: 1 objection(s). One objection from this angle. I know of no contrib module or extension that makes archive-fed logical decoding survive catalog conflicts, and no hook would let an extension do it outside core.
- demand: 1 objection(s). The GUC is off by default and the cost of the off path is one boolean check, so non-users pay little. The demand objection is a not-now point, not a rejection.

### Precedents retrieved

- P1: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P2: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - used in the review
- P3: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - used in the review; reason summarised from its thread
- P4: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P5: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P6: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P8: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant; reason summarised from its thread
- P9: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P10: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P11: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - used in the review; reason summarised from its thread
- P12: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - used in the review; reason summarised from its thread
- P13: [Minimal logical decoding on standbys (take 6)](https://commitfest.postgresql.org/patch/2968/) (cf, Returned with feedback, 2022-02-07) - used in the review
- P14: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P15: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P16: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P17: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P18: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant
- P19: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P20: [pg_walcleaner - new tool to detect, archive and delete the unneeded wal files](https://commitfest.postgresql.org/patch/3667/) (cf, Withdrawn, 2022-08-16) - judged not relevant
- P21: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P22: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P23: [Add recovery_min_apply_delay_reconnect recovery option](https://commitfest.postgresql.org/patch/1325/) (cf, Returned with feedback, 2017-11-15) - judged not relevant
- P24: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 200 s; cost reported by the CLI: $2.68
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 110 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:21:35.967Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
