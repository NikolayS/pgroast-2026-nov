# Design pre-review: Introduce XID age based replication slot invalidation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7103](https://commitfest.postgresql.org/patch/7103/) (Needs review); authors: Bharath Rupireddy (bharathrupireddy)
- Thread: [Introduce XID age based replication slot invalidation](https://www.postgresql.org/message-id/flat/CA%2B-JvFsMHckBMzsu5Ov9HCG3AFbMh056hHy1FiXazBRtZ9pFBg%40mail.gmail.com), started 2025-09-18
- Messages read: 65 of 65
- Patch prose read: 0045-Add-XID-age-based-replication-slot-invalidation.patch, 0044-Add-XID-age-based-replication-slot-invalidation.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is accepted in the thread and no objection at level 1a or 1b survived. Level 2 fails on four reproduced or unanswered design problems: replication-ondisk-1, replication-ondisk-4, replication-ondisk-2 and replication-ondisk-3. Their fixes exist only as descriptions in messages #62 to #64, not in a posted patch. maintenance-fit-1 adds that each slot owner needed its own handling and the design gives no general rule.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only the prose of the first two patch versions was read, not the v15 code. The cost of the per-relation vacuum check and the exact behaviour of the fixes proposed in messages #62 to #64 could not be judged.

## The proposal, restated

**Problem.** An idle, forgotten or lagging replication slot holds xmin or catalog_xmin back, so vacuum cannot freeze and the cluster moves toward XID wraparound. max_slot_wal_keep_size covers WAL, and idle_replication_slot_timeout is hard to set across a fleet with different XID rates. The target is operators of many clusters who want write availability over keeping slots.

**Proposed design.** A new GUC max_slot_xid_age invalidates a slot when its effective xmin or catalog_xmin is older than the limit. The checkpointer and restartpoint check all slots, and vacuum checks per relation in heap_vacuum_rel, but only unheld slots and only when a slot holds OldestXmin. v15 has three patches: the checkpoint path, the vacuum path, and invalidation of synced slots on standbys.

**User-visible surface.**
- GUC max_slot_xid_age (integer, 0 disables, max equal to vacuum_failsafe_age, postgresql.conf or command line only).
- New invalidation_reason value xid_aged in pg_replication_slots.
- New log messages for xid_aged invalidation, with the slot's xmin or catalog_xmin age.
- pg_createsubscriber disables XID-age invalidation during its run, like the idle timeout.
- Proposed changes to how the pg_conflict_detection slot and hot_standby_feedback catalog_xmin respond to the limit.

**What the author claims.**
- Invalidating a slot whose xmin is older than the limit lets vacuum freeze and avoids wraparound without manual vacuum or downtime.
- Setting the GUC at or just below vacuum_failsafe_age (1.6 billion) is the intended use.
- The per-relation check costs little because it reuses OldestXmin and slot horizons from one ComputeXidHorizons call.
- Vacuum never waits on a held slot, and the checkpointer invalidates held slots on its regular pass.
- Invalidated slots behave like those of other invalidation causes; the feature is off by default.

**Thread state.** Sawada, Bertrand, Amit, Nisha, Kuroda and others reviewed the patch in depth, and v15 needs a rebase. Open items are the pg_conflict_detection slot, standby physical slots invalidated through hot_standby_feedback, and slots invalidated while a long transaction holds the horizon.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-09-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Primary invalidates a healthy standby's physical slot through forwarded catalog_xmin

fixable · level 2, rubric item 7 · WAL / replication / standby, backpatchability · `replication-ondisk-1`

An operator sets max_slot_xid_age on the primary, and a failover subscription is disabled. hot_standby_feedback copies the synced slot's old catalog_xmin onto the standby's physical slot, and the next checkpoint invalidates that slot. The standby then loops on "can no longer access replication slot" and may need a rebuild once WAL is gone.

Message #60 reproduced this on v15, with the physical slot invalidated while its xmin age was 0. With the slot in synchronized_standby_slots, other failover slots also stall. The fix proposed in message #63 changes what hot_standby_feedback means: the primary would clamp the received catalog_xmin, and standby slots would fail later as rows_removed. That fix is not yet in a posted patch, and it changes behaviour that users of hot_standby_feedback depend on today.

Thread: [#60 (Nisha Moond, 2026-10-06)](https://www.postgresql.org/message-id/CABdArM48J96RfZYnfwWr-ptt4UsEAV8fjFNWxV22PcWp%2Bs-cgQ%40mail.gmail.com), [#63 (Bharath Rupireddy, 2026-10-21)](https://www.postgresql.org/message-id/CALj2ACU3mmAseGFHFJ0hzyJvgUJWsjbgMpG%2BhS8SObV40NAK9g%40mail.gmail.com)

**Question:** With the feedback clamp in place, what does a standby operator see and log, and how do they tell the primary's max_slot_xid_age was the cause?

Counter-check: held. Message #60 reproduced the physical slot invalidation on v15, and the clamp agreed in message #63 is not in a posted patch. Message #60 itself notes the misleading rows_removed reason on the standby, and no message resolves it.

### 2. Healthy logical slots invalidated when a transaction holds the horizon

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-4`

A long open transaction, or a repeatable read snapshot without an XID, pins catalog_xmin on the publisher. The checkpoint invalidates every caught-up logical slot, and their subscriptions break. Vacuum gains nothing, because the transaction still holds the same cutoff.

Message #61 shows this on v15 with an active, caught-up slot. Message #62 proposes a new proc-array scan in the checkpointer, with a database OID and KnownAssignedXids handling on standbys. That scan exists only as a description. Until it lands, the checkpoint path can break replication for a cause that slot invalidation cannot fix.

Thread: [#61 (Nisha Moond, 2026-10-07)](https://www.postgresql.org/message-id/CABdArM6RtWPFHiZZZV71Pvu_b7_yApyT7YGCRgCubm-95Sc74A%40mail.gmail.com), [#62 (Bharath Rupireddy, 2026-10-21)](https://www.postgresql.org/message-id/CALj2ACXUBz0dkMAjc9286D1RLSMjsPhsHjZWy%3DE%2B%3DnpQFdHxyw%40mail.gmail.com)

**Question:** How does the checkpoint path prove a slot is the actual horizon holder, on primary and standby, before it invalidates the slot?

Counter-check: held. Message #61 reproduced the invalidation of an active, caught-up slot with no vacuum gain, and the check proposed in message #62 is not in a patch. The snapshot-only case in #62 shows no vacuum gain, but it does not show pinned caught-up slots.

### 3. Launcher retention flag for pg_conflict_detection is not crash-safe

fixable · level 2, rubric item 14 · catalog changes, WAL / replication / standby · `replication-ondisk-2`

The launcher stops counting a disabled subscription and lets the slot's xmin advance, but subretentionactive stays true in pg_subscription. The server restarts with the new xmin already persisted. When the subscription is enabled, its apply worker believes retention was continuous and runs conflict detection over tuples that vacuum has already removed.

Message #64 keeps the stop decision in launcher shared memory because the launcher cannot update pg_subscription. Message #65 asks what happens across a postmaster restart, and the thread has no answer yet. The design also stops retention for all subscriptions, while message #57 and message #58 agreed to stop only the subscriptions holding the xmin. The persisted slot xmin and the catalog flag can disagree, and no step in the design brings them back in line.

Thread: [#57 (Bharath Rupireddy, 2026-09-18)](https://www.postgresql.org/message-id/CALj2ACXw3bStSS1UxTBP8vS0oOYVpO2Z0kivkaDdcHFBgomZLw%40mail.gmail.com), [#58 (Bertrand Drouvot, 2026-09-22)](https://www.postgresql.org/message-id/arJIJ3MbrWD6uKXG%40bdtpg), [#64 (Bharath Rupireddy, 2026-10-21)](https://www.postgresql.org/message-id/CALj2ACVvjs_1wWN6uHmMo3moRGSzWyuDqeZfR1WXjLXjxu30UQ%40mail.gmail.com), [#65 (Bertrand Drouvot, 2026-10-30)](https://www.postgresql.org/message-id/auSpz4wdC7PM9aWs%40bdtpg)

**Question:** What makes the slot's persisted xmin and subretentionactive agree after a crash between the launcher's decision and the apply worker's catalog update?

Counter-check: held. Message #65 asks what happens when the cleared slot xmin is persisted and the postmaster restarts while subretentionactive is still true. The thread ends there without an answer, so the design for pg_conflict_detection is open.

### 4. Synced slots churn and are lost at promotion

fixable · level 2, rubric item 14 · WAL / replication / standby · `replication-ondisk-3`

sync_replication_slots is on, and the primary's failover slot is idle. The standby's restartpoint invalidates the synced slot, and slotsync recreates it as temporary with a newer catalog_xmin, so it never syncs again. The checkpointer then SIGTERMs the slot sync worker about every max_slot_xid_age XIDs, and the slot is gone after promotion.

Message #53 reproduced this on v14, including the xid_aged reason disappearing so nothing remains to alert on. Message #54 and message #55 accept it as matching max_slot_wal_keep_size, and v15 only updates a comment (message #59). This feature targets slots whose horizon does not advance, so the stuck case is its normal case. For max_slot_wal_keep_size the same case is a rare corner.

Thread: [#53 (surya poondla, 2026-09-02)](https://www.postgresql.org/message-id/CAOVWO5pFay9VHzNbV-RVaZFCC%3DaJZuZv%2Bh0eHUSEfh%2B2hfQ1yA%40mail.gmail.com), [#54 (Nisha Moond, 2026-09-04)](https://www.postgresql.org/message-id/CABdArM7M5%3Daa8DCxeGNR8-BuR8u8V7sFEdBmBgRbaZvvfcdN-g%40mail.gmail.com), [#55 (surya poondla, 2026-09-15)](https://www.postgresql.org/message-id/CAOVWO5rszWY5%3Ds7b2w-tvtdWdEnWX8qB_Bg2RuxORvVSKEhKsg%40mail.gmail.com), [#56 (Nisha Moond, 2026-09-16)](https://www.postgresql.org/message-id/CABdArM5zjth_zzD_bYnZbMwm3svCndZu8hB8%2Bpd5O%3Ds-PRVN0g%40mail.gmail.com), [#59 (Bharath Rupireddy, 2026-09-23)](https://www.postgresql.org/message-id/CALj2ACVRpA1UnZx%3Ds0Ljsn01FcsEHEfj1mQXvQn69eO3Nq2YOA%40mail.gmail.com)

**Question:** After a synced slot is invalidated for XID age, how does the operator learn a failover slot is missing before promotion, and how is the periodic worker termination stopped?

Counter-check: held. Message #54 argues that max_slot_wal_keep_size has the same behaviour, but that case was not reproduced. This feature targets slots whose horizon does not advance, so the stuck slot and the repeated slot sync worker termination from message #53 are its expected case.

### 5. Each slot-owning subsystem needs its own xid_aged handling

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby, ABI / extension API · `maintenance-fit-1`

A later release adds a new internal slot owner, such as another launcher-managed or temporary decoding slot. Nobody adds an xid_aged exception or back-off path for it, so it is treated as an ordinary slot. Months later an operator with max_slot_xid_age set finds that feature broken or looping after a checkpoint, with only an invalidation LOG line.

Most open work in the thread is per-consumer special cases, not the core check. Message #53 reports a churn loop for synced slots. Message #57 and message #64 give pg_conflict_detection a launcher shmem flag protocol, and message #63 adds a feedback clamp for hot_standby_feedback. The design gives later slot owners no rule to follow and no check that fails when they ignore one, so the rule has to be relearned per feature (message #59 tested repack slots by hand).

Thread: [#47 (Bertrand Drouvot, 2026-08-25)](https://www.postgresql.org/message-id/ao1LFb96ELYtONpC%40bdtpg), [#53 (surya poondla, 2026-09-02)](https://www.postgresql.org/message-id/CAOVWO5pFay9VHzNbV-RVaZFCC%3DaJZuZv%2Bh0eHUSEfh%2B2hfQ1yA%40mail.gmail.com), [#57 (Bharath Rupireddy, 2026-09-18)](https://www.postgresql.org/message-id/CALj2ACXw3bStSS1UxTBP8vS0oOYVpO2Z0kivkaDdcHFBgomZLw%40mail.gmail.com), [#59 (Bharath Rupireddy, 2026-09-23)](https://www.postgresql.org/message-id/CALj2ACVRpA1UnZx%3Ds0Ljsn01FcsEHEfj1mQXvQn69eO3Nq2YOA%40mail.gmail.com), [#63 (Bharath Rupireddy, 2026-10-21)](https://www.postgresql.org/message-id/CALj2ACU3mmAseGFHFJ0hzyJvgUJWsjbgMpG%2BhS8SObV40NAK9g%40mail.gmail.com), [#64 (Bharath Rupireddy, 2026-10-21)](https://www.postgresql.org/message-id/CALj2ACVvjs_1wWN6uHmMo3moRGSzWyuDqeZfR1WXjLXjxu30UQ%40mail.gmail.com)

**Question:** What general rule tells a future slot owner how to react to xid_aged, and what would fail in tests if a new slot type did not follow it?

Counter-check: held. The thread shows three slot owners that each needed their own handling: synced slots, pg_conflict_detection and physical slots fed by hot_standby_feedback. Review found each one, and the proposal states no general rule or test that would catch the next one.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch invalidates replication slots in the checkpointer, restartpoint and vacuum, and affects synced slots and hot_standby_feedback. |
| wire protocol | no |  |
| catalog changes | **yes** | pg_replication_slots gains invalidation_reason xid_aged, and the pg_conflict_detection design interacts with subretentionactive in pg_subscription. |
| ABI / extension API | **yes** | src/include/replication/slot.h changes, which adds a new invalidation cause visible to code that uses slots. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds max_slot_xid_age, off by default, capped at vacuum_failsafe_age. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection replication-ondisk-1: Primary invalidates a healthy standby's physical slot through forwarded catalog_xmin |
| maintenance burden ("who maintains this in 10 years") | **yes** | Synced slots, pg_conflict_detection and hot_standby_feedback each need their own xid_aged handling, and later slot owners will too. |
| performance on the hot path | **yes** | Vacuum checks slots per relation in heap_vacuum_rel; the thread gives no numbers for many slots or many tables. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With the feedback clamp from message #63, what does a standby operator see in pg_replication_slots and the logs that points to the primary's max_slot_xid_age?
2. How does the checkpoint path prove, on primary and standby, that a slot and not a backend holds the horizon before it invalidates the slot?
3. After a restart between the launcher's stop decision and the apply worker's catalog update, what makes the slot's persisted xmin and subretentionactive agree?
4. After a synced slot is invalidated for XID age, how does the operator learn before promotion that a failover slot is missing?
5. What general rule tells a future slot owner how to react to xid_aged, and which test fails if a new slot type ignores it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The review notes say Sawada co-wrote much of the design and Amit is engaged. The owner must know slot invalidation, slot sync and the vacuum horizon code.
- **Liability:** The owner signs for every case where a slot is invalidated that should not be, across physical, logical, synced and conflict-detection slots, and for the changed hot_standby_feedback behaviour, for as long as the GUC exists.
- **Author follow-up:** The patch reached v15 over more than a year with responses to review, and the CommitFest entry was created in August 2026. v15 needs a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Operators of slot-heavy fleets gain from it and the thread accepts the motivation. Four reproduced or open design problems have fixes only as descriptions, and they reach into the checkpointer, vacuum, slot sync, the launcher and feedback handling.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Earlier objection on choosing the cutoff age is not yet answered. Refuted (already handled): Message #20 answers the cutoff question with a concrete value tied to vacuum_failsafe_age, and message #24 caps the GUC there. The low-value scenario costs only an operator who ignores that guidance, and the long-transaction part duplicates replication-ondisk-4.
- `standards-compat-1` Primary-side GUC quietly changes what hot_standby_feedback guarantees. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same feedback clamp and the same misleading rows_removed reason on the standby.
- `standards-compat-2` Server GUC overrides per-subscription retain_dead_tuples without durable state. Refuted (not specific): Duplicate of replication-ondisk-2, which states the same restart gap between the launcher's shared-memory flag and subretentionactive more directly.
- `security-1` Any session holding an old snapshot can get healthy slots invalidated. Refuted (not specific): Duplicate of replication-ondisk-4, with the same failure and the same required fix. Message #61 also ties the pinned catalog_xmin to a transaction that has an XID, so the snapshot-only variant is not shown to pin caught-up slots.
- `maintenance-fit-2` Fix for standby feedback changes hot_standby_feedback semantics on the primary. Refuted (not specific): Duplicate of replication-ondisk-1, which already covers the feedback clamp, its absence from v15 and the wrong cause shown on the standby.
- `already-exists-1` Age-based slot cleanup is already possible with a monitoring query and pg_drop_replication_slot. Refuted (already handled): The objection itself concedes that an external job cannot drop synced slots on standbys. Core already has max_slot_wal_keep_size and idle_replication_slot_timeout for the same class of problem, and reviewers from several companies accepted the motivation in messages #2 and #8.

### Review angles

- prior-attempts: 1 objection(s). [P4] concerns rejecting already invalidated physical slots. Its backward-compatibility reason does not apply to an opt-in GUC that is off by default.
- standards-compat: 2 objection(s). The GUC itself follows the existing pattern of max_slot_wal_keep_size and idle_replication_slot_timeout, adds no syntax, and is off by default. The synced-slot recreation behaviour also matches the existing WAL case (messages #54 to #56).
- replication-ondisk: 4 objection(s). No on-disk format, WAL record, or catalog column change is described beyond a new invalidation_reason value, so pg_upgrade is not affected.
- security: 1 objection(s). Only superusers or the server config can set the GUC. The new view value and log lines expose slot xmin ages that pg_replication_slots already shows. I found no other privilege exposure.
- maintenance-fit: 2 objection(s). Hot-path cost and ownership were checked and give no objection. The vacuum check reuses horizons from one ComputeXidHorizons call and runs only when a slot holds OldestXmin. Sawada, a committer who knows the area, co-wrote much of the design, and Amit is engaged.
- already-exists: 1 objection(s). Only one objection from this angle. The feature extends the existing slot invalidation mechanism and does not duplicate it.
- demand: 0 objection(s). The material shows enough demand. Slots that hold xmin or catalog_xmin toward wraparound are a known failure, and reviewers from several companies (messages #2, #8, #16, #38) accept the motivation. Costs fall mainly on users who set the GUC, since it is off by default. A monitoring query plus a script that drops slots is a workaround, but it needs every operator to build the same safety net against wraparound, so I do not raise 'not now, show demand'.

### Precedents retrieved

- P1: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P2: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P3: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P4: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P5: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P6: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P7: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P8: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P9: [Autovacuum: vacuum tables in descending order of xid_age](https://commitfest.postgresql.org/patch/2364/) (cf, Returned with feedback, 2020-08-03) - judged not relevant; reason summarised from its thread
- P10: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P11: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P12: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P13: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P14: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P15: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P16: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P17: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P18: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P19: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P20: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P21: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P22: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P23: [64-bit XIDs](https://commitfest.postgresql.org/patch/3594/) (cf, Returned with feedback, 2023-12-15) - judged not relevant
- P24: [Reset vacuum_defer_cleanup_age to PGC_SIGHUP. Revert commit 84725aa5efe11688633b553e58113efce4181f2e](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f480e294498533820f3ef3e6de4dcb8ff5401140) (revert, Partially reverted, 2013-02-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 151 s; cost reported by the CLI: $5.36
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 103 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:19:44.696Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
