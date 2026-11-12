# Design pre-review: Introduce XID age based replication slot invalidation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7103](https://commitfest.postgresql.org/patch/7103/); [thread](https://www.postgresql.org/message-id/flat/CA%2B-JvFsMHckBMzsu5Ov9HCG3AFbMh056hHy1FiXazBRtZ9pFBg%40mail.gmail.com), 65 of 65 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is accepted in the thread and no objection at level 1a or 1b survived. Level 2 fails on four reproduced or unanswered design problems: replication-ondisk-1, replication-ondisk-4, replication-ondisk-2 and replication-ondisk-3. Their fixes exist only as descriptions in messages #62 to #64, not in a posted patch. maintenance-fit-1 adds that each slot owner needed its own handling and the design gives no general rule.

Not checked: Only the prose of the first two patch versions was read, not the v15 code. The cost of the per-relation vacuum check and the exact behaviour of the fixes proposed in messages #62 to #64 could not be judged.

## Prior attempts

None found before 2025-09-18.

## Objections a reviewer is likely to raise

### 1. Primary invalidates a healthy standby's physical slot through forwarded catalog_xmin

An operator sets max_slot_xid_age on the primary, and a failover subscription is disabled. hot_standby_feedback copies the synced slot's old catalog_xmin onto the standby's physical slot, and the next checkpoint invalidates that slot. The standby then loops on "can no longer access replication slot" and may need a rebuild once WAL is gone.

Message #60 reproduced this on v15, with the physical slot invalidated while its xmin age was 0. With the slot in synchronized_standby_slots, other failover slots also stall. The fix proposed in message #63 changes what hot_standby_feedback means: the primary would clamp the received catalog_xmin, and standby slots would fail later as rows_removed. That fix is not yet in a posted patch, and it changes behaviour that users of hot_standby_feedback depend on today.

**Question:** With the feedback clamp in place, what does a standby operator see and log, and how do they tell the primary's max_slot_xid_age was the cause?

<sub>fixable, level 2, WAL / replication / standby; messages [#60](https://www.postgresql.org/message-id/CABdArM48J96RfZYnfwWr-ptt4UsEAV8fjFNWxV22PcWp%2Bs-cgQ%40mail.gmail.com), [#63](https://www.postgresql.org/message-id/CALj2ACU3mmAseGFHFJ0hzyJvgUJWsjbgMpG%2BhS8SObV40NAK9g%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Healthy logical slots invalidated when a transaction holds the horizon

A long open transaction, or a repeatable read snapshot without an XID, pins catalog_xmin on the publisher. The checkpoint invalidates every caught-up logical slot, and their subscriptions break. Vacuum gains nothing, because the transaction still holds the same cutoff.

Message #61 shows this on v15 with an active, caught-up slot. Message #62 proposes a new proc-array scan in the checkpointer, with a database OID and KnownAssignedXids handling on standbys. That scan exists only as a description. Until it lands, the checkpoint path can break replication for a cause that slot invalidation cannot fix.

**Question:** How does the checkpoint path prove a slot is the actual horizon holder, on primary and standby, before it invalidates the slot?

<sub>fixable, level 2, WAL / replication / standby; messages [#61](https://www.postgresql.org/message-id/CABdArM6RtWPFHiZZZV71Pvu_b7_yApyT7YGCRgCubm-95Sc74A%40mail.gmail.com), [#62](https://www.postgresql.org/message-id/CALj2ACXUBz0dkMAjc9286D1RLSMjsPhsHjZWy%3DE%2B%3DnpQFdHxyw%40mail.gmail.com); `replication-ondisk-4`</sub>

### 3. Launcher retention flag for pg_conflict_detection is not crash-safe

The launcher stops counting a disabled subscription and lets the slot's xmin advance, but subretentionactive stays true in pg_subscription. The server restarts with the new xmin already persisted. When the subscription is enabled, its apply worker believes retention was continuous and runs conflict detection over tuples that vacuum has already removed.

Message #64 keeps the stop decision in launcher shared memory because the launcher cannot update pg_subscription. Message #65 asks what happens across a postmaster restart, and the thread has no answer yet. The design also stops retention for all subscriptions, while message #57 and message #58 agreed to stop only the subscriptions holding the xmin. The persisted slot xmin and the catalog flag can disagree, and no step in the design brings them back in line.

**Question:** What makes the slot's persisted xmin and subretentionactive agree after a crash between the launcher's decision and the apply worker's catalog update?

<sub>fixable, level 2, catalog changes; messages [#57](https://www.postgresql.org/message-id/CALj2ACXw3bStSS1UxTBP8vS0oOYVpO2Z0kivkaDdcHFBgomZLw%40mail.gmail.com), [#58](https://www.postgresql.org/message-id/arJIJ3MbrWD6uKXG%40bdtpg), [#64](https://www.postgresql.org/message-id/CALj2ACVvjs_1wWN6uHmMo3moRGSzWyuDqeZfR1WXjLXjxu30UQ%40mail.gmail.com), [#65](https://www.postgresql.org/message-id/auSpz4wdC7PM9aWs%40bdtpg); `replication-ondisk-2`</sub>

### 4. Synced slots churn and are lost at promotion

sync_replication_slots is on, and the primary's failover slot is idle. The standby's restartpoint invalidates the synced slot, and slotsync recreates it as temporary with a newer catalog_xmin, so it never syncs again. The checkpointer then SIGTERMs the slot sync worker about every max_slot_xid_age XIDs, and the slot is gone after promotion.

Message #53 reproduced this on v14, including the xid_aged reason disappearing so nothing remains to alert on. Message #54 and message #55 accept it as matching max_slot_wal_keep_size, and v15 only updates a comment (message #59). This feature targets slots whose horizon does not advance, so the stuck case is its normal case. For max_slot_wal_keep_size the same case is a rare corner.

**Question:** After a synced slot is invalidated for XID age, how does the operator learn a failover slot is missing before promotion, and how is the periodic worker termination stopped?

<sub>fixable, level 2, WAL / replication / standby; messages [#53](https://www.postgresql.org/message-id/CAOVWO5pFay9VHzNbV-RVaZFCC%3DaJZuZv%2Bh0eHUSEfh%2B2hfQ1yA%40mail.gmail.com), [#54](https://www.postgresql.org/message-id/CABdArM7M5%3Daa8DCxeGNR8-BuR8u8V7sFEdBmBgRbaZvvfcdN-g%40mail.gmail.com), [#55](https://www.postgresql.org/message-id/CAOVWO5rszWY5%3Ds7b2w-tvtdWdEnWX8qB_Bg2RuxORvVSKEhKsg%40mail.gmail.com), [#56](https://www.postgresql.org/message-id/CABdArM5zjth_zzD_bYnZbMwm3svCndZu8hB8%2Bpd5O%3Ds-PRVN0g%40mail.gmail.com), [#59](https://www.postgresql.org/message-id/CALj2ACVRpA1UnZx%3Ds0Ljsn01FcsEHEfj1mQXvQn69eO3Nq2YOA%40mail.gmail.com); `replication-ondisk-3`</sub>

### 5. Each slot-owning subsystem needs its own xid_aged handling

A later release adds a new internal slot owner, such as another launcher-managed or temporary decoding slot. Nobody adds an xid_aged exception or back-off path for it, so it is treated as an ordinary slot. Months later an operator with max_slot_xid_age set finds that feature broken or looping after a checkpoint, with only an invalidation LOG line.

Most open work in the thread is per-consumer special cases, not the core check. Message #53 reports a churn loop for synced slots. Message #57 and message #64 give pg_conflict_detection a launcher shmem flag protocol, and message #63 adds a feedback clamp for hot_standby_feedback. The design gives later slot owners no rule to follow and no check that fails when they ignore one, so the rule has to be relearned per feature (message #59 tested repack slots by hand).

**Question:** What general rule tells a future slot owner how to react to xid_aged, and what would fail in tests if a new slot type did not follow it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#47](https://www.postgresql.org/message-id/ao1LFb96ELYtONpC%40bdtpg), [#53](https://www.postgresql.org/message-id/CAOVWO5pFay9VHzNbV-RVaZFCC%3DaJZuZv%2Bh0eHUSEfh%2B2hfQ1yA%40mail.gmail.com), [#57](https://www.postgresql.org/message-id/CALj2ACXw3bStSS1UxTBP8vS0oOYVpO2Z0kivkaDdcHFBgomZLw%40mail.gmail.com), [#59](https://www.postgresql.org/message-id/CALj2ACVRpA1UnZx%3Ds0Ljsn01FcsEHEfj1mQXvQn69eO3Nq2YOA%40mail.gmail.com), [#63](https://www.postgresql.org/message-id/CALj2ACU3mmAseGFHFJ0hzyJvgUJWsjbgMpG%2BhS8SObV40NAK9g%40mail.gmail.com), [#64](https://www.postgresql.org/message-id/CALj2ACVvjs_1wWN6uHmMo3moRGSzWyuDqeZfR1WXjLXjxu30UQ%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. With the feedback clamp from message #63, what does a standby operator see in pg_replication_slots and the logs that points to the primary's max_slot_xid_age?
2. How does the checkpoint path prove, on primary and standby, that a slot and not a backend holds the horizon before it invalidates the slot?
3. After a restart between the launcher's stop decision and the apply worker's catalog update, what makes the slot's persisted xmin and subretentionactive agree?
4. After a synced slot is invalidated for XID age, how does the operator learn before promotion that a failover slot is missing?
5. What general rule tells a future slot owner how to react to xid_aged, and which test fails if a new slot type ignores it?

## Touch points

- **WAL / replication / standby:** The patch invalidates replication slots in the checkpointer, restartpoint and vacuum, and affects synced slots and hot_standby_feedback.
- **catalog changes:** pg_replication_slots gains invalidation_reason xid_aged, and the pg_conflict_detection design interacts with subretentionactive in pg_subscription.
- **ABI / extension API:** src/include/replication/slot.h changes, which adds a new invalidation cause visible to code that uses slots.
- **new GUC:** The patch adds max_slot_xid_age, off by default, capped at vacuum_failsafe_age.
- **backpatchability:** see objection replication-ondisk-1: Primary invalidates a healthy standby's physical slot through forwarded catalog_xmin
- **maintenance burden ("who maintains this in 10 years"):** Synced slots, pg_conflict_detection and hot_standby_feedback each need their own xid_aged handling, and later slot owners will too.
- **performance on the hot path:** Vacuum checks slots per relation in heap_vacuum_rel; the thread gives no numbers for many slots or many tables.

## Who would commit it, and when

- Owner: qualified committer engaged. The review notes say Sawada co-wrote much of the design and Amit is engaged. The owner must know slot invalidation, slot sync and the vacuum horizon code.
- Priority: fix design first; audience a sizeable minority; review cost high. Operators of slot-heavy fleets gain from it and the thread accepts the motivation. Four reproduced or open design problems have fixes only as descriptions, and they reach into the checkpointer, vacuum, slot sync, the launcher and feedback handling.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
