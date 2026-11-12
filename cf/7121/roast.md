# Design pre-review: Archive-fed logical decoding: pausing recovery on slot conflict

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7121](https://commitfest.postgresql.org/patch/7121/); [thread](https://www.postgresql.org/message-id/flat/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is plausible and a reviewer supports it, but the design is not settled. v2 admits that some pauses cannot resolve (replication-ondisk-1) and nothing bounds them (standards-compat-1). The per-slot question from message #1 is still open (standards-compat-3), and the proposal does not say how the wait inside redo handles reload and shutdown (prior-attempts-1). These are fixable, but the list needs to agree on them before code review.

Not checked: Without the code, the review could not check how the wait loop handles interrupts, or whether the pause and auto-resume are free of races with slot invalidation and concurrent decoding. It also could not check the claim of one boolean test with the GUC off.

## Prior attempts

- [Use latch instead of pg_usleep for standby recovery conflicts](https://www.postgresql.org/message-id/flat/CANP8+j+WWawZCR+w=887FcnqQibFf=j=kag98XNN9rAOM-4HTg@mail.gmail.com) (related; Returned with feedback, 2017-04-03). "Calling HandleStartupProcInterrupts() is actually dangerous I think, because it would reload the GUC context while replaying a record." ([message](https://www.postgresql.org/message-id/CAB7nPqSMu1BAPgHK2aqmYm%3D7f__VzwvXDCVHqYLZzbH1HFbE-Q%40mail.gmail.com)) Different now: P12 changed how an existing conflict wait sleeps. This proposal adds a new wait at the same place inside record replay, so P12's concern about reloading configuration in the middle of a record applies to it.

## Objections a reviewer is likely to raise

### 1. Some pauses cannot resolve because of WAL ordering, and nothing bounds them

On the primary, a catalog row deleted by xid X is pruned on access soon after X commits, before the next xl_running_xacts is logged. The standby pauses before the prune record. Decoding can read only up to the replay position, so it never sees a running_xacts newer than X, and the standby waits until someone notices.

Message #2 and the docs say a pause may not be resolvable, but the patch does not say how often this happens or why. Decoding gets a new catalog_xmin candidate from xl_running_xacts records. Any running_xacts that would move the horizon past the prune horizon may sit after the pause LSN, where decoding cannot reach it. That would make the stall a regular event under catalog churn, and the auto-resume promise that message #1 makes depends on how often it happens.

**Question:** Under realistic catalog churn, what fraction of conflict pauses can decoding resolve from WAL already replayed, and what ends the others without an operator?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Unbounded conflict wait departs from the max_standby_*_delay pattern

The pg_recvlogical process on an archive-fed standby crashes overnight. Recovery pauses at the next catalog prune record and stays paused, while cascading standbys and readers see data that keeps getting older. Monitoring that checks only for invalidated slots or walreceiver state reports nothing wrong.

Every existing recovery conflict wait in Postgres is bounded by max_standby_archive_delay or max_standby_streaming_delay, and waiting forever requires the explicit value -1. The proposal adds a conflict wait with no bound, controlled by an on/off boolean. Message #3 asks for a maximum pause duration and repeated warnings, and v2 gives no answer. If a delay setting is added later, the boolean stays behind as a second knob that cannot be removed.

**Question:** Why should this conflict wait be an unbounded boolean, when other recovery conflict waits use a delay setting where -1 means wait forever?

<sub>fixable, level 2, new GUC; messages [#2](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Cluster-wide GUC for behaviour that slot properties already express

A standby serves as a failover target and also hosts a CDC slot and a test slot that someone left behind. With the GUC on, the abandoned test slot stalls replay for the whole standby. With the GUC off, the CDC slot is invalidated. No setting protects only the CDC slot.

Per-slot behaviour such as failover and two_phase is set as a slot property at creation. Here a server-wide GUC decides how every local logical slot reacts to conflicts. Message #1 asks the same question (item 2) and calls the GUC coarse, and v2 keeps the GUC without saying why. If a per-slot option is added later, the GUC becomes a lasting second interface.

**Question:** Why is a server-wide GUC preferable to a per-slot option, given that the thread itself raised the per-slot option and existing slot behaviours are set per slot?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/7781AD24-1714-479C-8269-DA34F5FFDD24%40yandex-team.ru); `standards-compat-3`</sub>

### 4. Unbounded wait inside record replay versus startup interrupt handling

A consumer dies and recovery pauses inside ResolveRecoveryConflictWithSnapshot() for hours. The operator sets recovery_pause_on_logical_slot_conflict = off and reloads, or runs pg_ctl stop. Either the startup process never handles the SIGHUP or shutdown request, or it handles it while a record is half replayed.

Existing recovery pauses happen between records in the main loop, and other conflict waits inside redo are bounded by max_standby_*_delay. This pause sits inside a record's redo and has no bound (message #3). In [P12], the recorded reason was that calling HandleStartupProcInterrupts() there would reload the GUC context while a record is being replayed. v1-0001 lists only ConfirmRecoveryPaused() and CheckForStandbyTrigger() for the wait loop, and does not say how config reload and shutdown are handled.

Precedent: [Use latch instead of pg_usleep for standby recovery conflicts](https://www.postgresql.org/message-id/flat/CANP8+j+WWawZCR+w=887FcnqQibFf=j=kag98XNN9rAOM-4HTg@mail.gmail.com) (Returned with feedback, 2017-04-03)

**Question:** How does the conflict wait handle SIGHUP and shutdown requests during a long pause, and why is that safe in the middle of replaying a record?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 5. Primary-side placeholder slot can already hold catalog_xmin without streaming

An operator who can run one SQL connection to the primary keeps a logical slot there. A small job advances it to the standby slot's restart_lsn, so the primary never prunes catalog rows the standby still needs. With this setup the core GUC adds nothing, and the stalls it causes on the archive-fed standby can be avoided.

Message #1, question 1, names an out-of-band channel that publishes the standby's catalog_xmin back to the primary as a possible long-term answer. Postgres can already build that channel with pg_create_logical_replication_slot and pg_replication_slot_advance on the primary. The author's goal is no load on, and no dependency on, the primary. That makes the existing way less convenient, because it keeps WAL on the primary and needs a connection, but it does not make it impossible. A committer would likely ask which users truly cannot open any connection to the primary.

**Question:** Which deployments cannot run a primary-side logical slot advanced by an external job, and why is a recovery pause on the standby better for them?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/967DBEA9-E8B7-4705-AD36-447D839AACA9%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAFPTHDa9GeU3TG4bAaDVj0XrZBQpcce5N1B5ziobP4sDuav%2B%3DQ%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Under realistic catalog churn, what fraction of conflict pauses can decoding resolve from WAL already replayed, and what ends the others without an operator?
2. Will the pause get a time limit as message #3 asks, and if so, why not a delay setting where -1 means wait forever, like max_standby_archive_delay?
3. Why is a server-wide GUC preferable to a per-slot option, given that an abandoned slot can stall replay for the slot the operator wants to protect?
4. How does the conflict wait handle SIGHUP and shutdown requests during a long pause, and why is that safe in the middle of replaying a record?
5. Which deployments cannot keep a logical slot on the primary advanced by an external job, and why is a recovery pause on the standby better for them?

## Touch points

- **WAL / replication / standby:** The patch adds a wait inside ResolveRecoveryConflictWithSnapshot() that pauses replay before a logical slot is invalidated.
- **new GUC:** The patch adds recovery_pause_on_logical_slot_conflict, a PGC_SIGHUP boolean that is off by default.
- **security / privileges:** With the GUC on, any role that owns an idle local logical slot can hold back replay for the whole standby.
- **maintenance burden ("who maintains this in 10 years"):** A second pause point inside record replay and a new pause cause must be kept working through later changes to recovery.
- **performance on the hot path:** The check runs on the replay path for snapshot conflicts; message #1 says the cost with the GUC off is one boolean test.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #1 asks whether an out-of-band feedback channel or a separate decoding-from-archive tool is the better layer.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the startup process, recovery conflict handling and logical slot invalidation on standbys. The visible thread shows none taking part in that role.
- Priority: fix design first; audience niche; review cost medium. Message #1 limits the use to dedicated decoding standbys fed by restore_command, and shows demand only from the authors' own field test. The patch is small but changes the startup process and slot invalidation, and the pause bound and per-slot questions are open.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
