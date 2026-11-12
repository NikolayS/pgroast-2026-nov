# Design pre-review: Issue with synced logical replication slot during switchover

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6132](https://commitfest.postgresql.org/patch/6132/) (Needs review); authors: Fabrice Chapuis (fabricec)
- Thread: [Issue with logical replication slot during switchover](https://www.postgresql.org/message-id/flat/CAA5-nLAqGpBFEAr2XNYMj3E%2B39caQra_SJeB5MCtp7PCyLTiOg%40mail.gmail.com), started 2025-08-07
- Messages read: 66 of 66
- Patch prose read: v1-0001-Resync-logical-replication-slot-after-switchover.patch, v1-0001-fix-failover-slot-issue-when-doing-a-switchover.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The entry does not name one design (maintenance-fit-3). The allow_overwrite series, the message #42 patch and the message #66 rule differ in on-disk and API cost. The demand is real and the goal is accepted in the thread. The reviewers still disagree on whether name plus failover=true is enough to overwrite a slot, so the list must settle the rule before code review makes sense.

First failing level, from the base up: **Level 1a: understandability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: failed
- Level 1b: desirability: not evaluated
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot tell whether the message #42 patch checks any local slot property before the drop. It also cannot tell whether the recreated slot becomes sync-ready in the same cycle.

## The proposal, restated

**Problem.** After a switchover, the old primary joins as a standby but keeps its logical slots with failover=true and synced=false. The slot sync worker then stops with "exiting from slot synchronization because same name slot ... already exists on the standby". HA tools such as Patroni must drop these slots and wait for an asynchronous resync, so for a while no standby may hold ready logical slots.

**Proposed design.** The latest patch (message #42) changes drop_local_obsolete_slots() and related helpers in slotsync.c. A local slot with failover=true and synced=false is treated as a sync candidate, then dropped and recreated from the primary's slot. An earlier series (v1 to v6) instead added an allow_overwrite slot property, set at creation and shown in pg_replication_slots.

**User-visible surface.**
- Changed slot sync behavior: a local slot with failover=true and synced=false no longer stops the slot sync worker and gets overwritten (latest patch).
- Earlier series: new allow_overwrite argument to pg_create_logical_replication_slot().
- Earlier series: new allow_overwrite column in pg_replication_slots.
- Earlier series: new allow_overwrite field in the on-disk slot data and a changed ReplicationSlotCreate() C signature.
- Discussed only: a pg_alter_logical_replication_slot() function to change slot properties after creation.

**What the author claims.**
- The failover flag alone shows that a slot was created on a primary and is meant to be synced, since logical failover slots cannot be created on a standby today.
- A second switchover already works, because the synced flag is then true, so the first-switchover failure is an inconsistency.
- pg_rewind removes pg_replslot, so failover=true and synced=false on a healthy standby can only follow a clean shutdown of the old primary (Kukushkin).
- An allow_overwrite flag would be set unconditionally in practice, so it adds no safety (Kukushkin).

**Thread state.** Shveta Malik, Amit Kapila and Masahiko Sawada disagree on whether name plus failover=true is enough to overwrite a slot. Amit asks for extra slot metadata or a user API, Sawada shows a cascading case where allow_overwrite fails, and no approach has consensus; the CF entry needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-08-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Two incompatible designs with no stated choice

fixable · level 1a, rubric item 1 · problem statement, on-disk format / pg_upgrade, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A committer picking up the CF entry cannot tell which design is proposed. The v6 series adds an allow_overwrite field to on-disk slot data and changes the ReplicationSlotCreate() signature that extensions call. The message #42 patch changes only sync behaviour, so the commit and pg_upgrade cost differ completely between the two.

The CF entry still points to the allow_overwrite series as reviewed in messages #28 to #31. The latest commit message does not say whether it replaces that series. Message #64 shows a cascading switchover where allow_overwrite blocks a wanted overwrite. Message #66 proposes a third rule based on synchronized_standby_slots and primary_conninfo, with no patch. A permanent on-disk field and ABI change should not land while the semantics are still moving.

Thread: [#31 (Fabrice Chapuis, 2025-10-15)](https://www.postgresql.org/message-id/CAA5-nLDXU3zVAKmeE5TYF0nyWjUDrxzK_2w5hmy0mgwqsEspRw%40mail.gmail.com), [#42 (Fabrice Chapuis, 2025-11-07)](https://www.postgresql.org/message-id/CAA5-nLDqGFgZhV1T2GwjRAAWA-b0oBuWCkd1g1g0Aup45Dpm8A%40mail.gmail.com), [#64 (Masahiko Sawada, 2025-11-20)](https://www.postgresql.org/message-id/CAD21AoDH%2BNv62qoJ8P%2B2249u%3DHTyrWg3qGuk9vqOb014%3DzzWzg%40mail.gmail.com), [#66 (Fabrice Chapuis, 2025-11-24)](https://www.postgresql.org/message-id/CAA5-nLCx5qL%3DGk7DYFmtyoOtASAEtsvDpo%3DZMyYFnuMv%3DqVFuQ%40mail.gmail.com)

**Question:** Which single design does the entry now propose, and does it change on-disk slot data or the ReplicationSlotCreate() signature?

Counter-check: held. Message #41 suggests the message #42 patch replaces the allow_overwrite series, but the author returns to allow_overwrite in message #63 and proposes a third rule in message #66. The entry does not name one design.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The earlier series adds an allow_overwrite field to the on-disk slot data. |
| WAL / replication / standby | **yes** | The latest patch changes when the slot sync worker drops and recreates local logical slots on a standby. |
| wire protocol | no |  |
| catalog changes | **yes** | The earlier series changes pg_proc.dat and system_functions.sql and adds a column to pg_replication_slots. |
| ABI / extension API | **yes** | The earlier series changes the ReplicationSlotCreate() C signature in slot.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CF entry is tagged Bugfix for a failure reported on 17.5, but the fix changes slot sync behavior. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The latest patch reads failover=true as slot provenance, which later changes to failover slot creation must recheck. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | HA tools can already drop the slot and call pg_sync_replication_slots(), as message #40 notes. |

## 5. Questions the author must answer before anyone reviews code

1. Which single design does the entry now propose: the allow_overwrite series, the message #42 patch, or the message #66 rule?
2. Does the chosen design change the on-disk slot data or the ReplicationSlotCreate() signature?
3. Which properties of the local slot (plugin, database, LSNs, catalog_xmin) does the patch compare with the remote slot before dropping it, and what happens when they differ?
4. What slot state, other than failover=true, would tell a former primary's slot apart once standbys may create failover slots?
5. How does the message #42 patch shorten the time without sync-ready slots compared with pg_drop_replication_slot() followed by pg_sync_replication_slots()?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Amit Kapila and Masahiko Sawada, who know slot sync, take part in the thread; neither has endorsed a design.
- **Liability:** The owner signs for a rule that lets the slot sync worker drop user-visible logical slots. With the allow_overwrite series, the owner also signs for a permanent on-disk field and a function argument.
- **Author follow-up:** The author posted patches and new proposals up to message #66, but the CF entry has needed a rebase since 2026-10-02.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Message #1 reports a production failure with Patroni, and a Patroni maintainer confirms the gap in message #36, so the goal is valid. The latest patch is small and touches only slotsync.c, but three candidate rules are open and none has consensus.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-1` (fixable, level 2) Name and failover flag alone authorize dropping a local slot. Not listed: level 2 is above the failing level 1a: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Failover flag reused as slot provenance blocks future standby-created failover slots. Not listed: level 2 is above the failing level 1a: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Old primary's failover slots are dropped automatically with no opt-in. Not listed: level 2 is above the failing level 1a: not evaluated (base-up stop rule).
- `already-exists-1` (fixable, level 1b) Latest patch automates a drop-and-resync that HA tools can already run. Not listed: level 1b is above the failing level 1a: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Latest patch returns to the failover-flag rule set aside in an earlier thread. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same rejoin-without-pg_rewind case and the missing checks more concretely. The earlier decision is known here only through the report in message #34, so it adds little support.
- `standards-compat-1` Silent drop of failover slots changes behavior on upgrade. Refuted (not specific): Duplicate of maintenance-fit-2, which covers the same different-upstream drop. The CF entry targets version 19, so the backpatch concern has no support, and the missing documentation hunk is outside levels 1-2.
- `standards-compat-2` Using the failover flag as overwrite permission locks in its meaning. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same future cascading-standby case and names the maintenance cost more concretely.
- `replication-ondisk-2` No persistent slot identity for future standby-created failover slots. Refuted (not specific): Duplicate of maintenance-fit-1. The CF entry targets version 19, so the back-branch question has no support in the material.
- `replication-ondisk-3` Drop-and-recreate may keep the window with no ready slot. Refuted (not specific): Duplicate of already-exists-1, which raises the same window without a sync-ready slot and also compares it with the drop that HA tools can do today.

### Review angles

- prior-attempts: 1 objection(s). P1 and P3 are earlier attempts at slot sync itself. One stalled on catalog_xmin safety and the other on a rebase, and neither reason applies to overwriting after a switchover. The only prior rejection that applies is the earlier thread named in message #34.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. pg_dump does not apply because replication slots are not dumped.
- replication-ondisk: 3 objection(s). The listed precedents cover the first slot sync designs. Their recorded reasons do not apply to overwriting slots after a switchover.
- security: 0 objection(s). From the security angle there is nothing to object to. Both designs act only on slots that need the REPLICATION attribute or superuser to create, run inside the slot sync worker, and give no new access to data, files or other sessions. The risk of overwriting an unrelated slot is a correctness and design question for other angles.
- maintenance-fit: 3 objection(s). Hot-path cost is not a concern: the change runs only in the slot sync worker. Amit Kapila and Masahiko Sawada, who know slot sync, are engaged in the thread, so ownership is not the weak point.
- already-exists: 1 objection(s). The allow_overwrite series (v1 to v6) adds a slot property that no existing feature offers, so this angle raises nothing against it.
- demand: 0 objection(s). The material shows real demand. Message #1 reports a production failure on Postgres 17.5 with Patroni during a routine switchover, and message #36 from a Patroni maintainer says the drop-and-resync workaround can leave no standby with ready logical slots if the primary fails during the resync window. Sawada (message #56) calls the reuse idea reasonable, so the open questions are about design, not demand.

### Precedents retrieved

- P1: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - used in the review; reason summarised from its thread
- P2: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P3: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - used in the review; reason summarised from its thread
- P4: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P5: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P6: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P7: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P8: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P9: [Making WAL receiver startup rely on GUC context for primary_conninfo and primary_slot_name](https://commitfest.postgresql.org/patch/1916/) (cf, Rejected, 2019-02-16) - judged not relevant; reason summarised from its thread
- P10: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P11: [Support multiple synchronous standby servers](https://commitfest.postgresql.org/patch/293/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P12: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant
- P15: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P16: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P17: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P18: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant
- P19: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P20: [PARALLEL flag for cursors](https://commitfest.postgresql.org/patch/1362/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P21: [deferred primary key and logical replication](https://commitfest.postgresql.org/patch/2631/) (cf, Returned with feedback, 2020-11-29) - judged not relevant
- P22: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant
- P23: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - judged not relevant
- P24: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 189 s; cost reported by the CLI: $4.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 101 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:43:18.417Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
