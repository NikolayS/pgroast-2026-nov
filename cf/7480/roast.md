# Design pre-review: walreceiver: stale hot_standby_feedback xmin on primary when wal_receiver_status_interval = 0 (PG16 and later)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7480](https://commitfest.postgresql.org/patch/7480/); [thread](https://www.postgresql.org/message-id/flat/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The bug is reproduced and the goal is not in dispute. The open point is which interval-0 behaviour back branches should get: replication-ondisk-1 shows that the patched build invalidates logical slots on a standby that work today, and standards-compat-2 shows the choice of "feedback off" over the 15 behaviour is not argued. maintenance-fit-2 adds that slotsync still accepts interval 0. The list should agree on the semantics before code review.

Not checked: The code and the TAP test were not read, so the stability of the test and the exact effect on primary_has_standby_xmin could not be checked. Message #4 is known only through the brief and the objections.

## Prior attempts

- [Forbid the use of invalidated physical slots in streaming replication.](https://www.postgresql.org/message-id/flat/OS0PR01MB5716A626A4AF5814E057CEE39484A@OS0PR01MB5716.jpnprd01.prod.outlook.com) (related; Returned with feedback, 2024-02-01). "I think making logical and physical slot behaviour consistent would be better but if the inconsistent behaviour is out there for some releases, changing that now will break backward compatibility." ([message](https://www.postgresql.org/message-id/CAExHW5t9w_iSU%2BVP-GqGLeRErRJnGzkqavxzYfCk8Z%2Be0pu-3w%40mail.gmail.com)) Different now: P8 changed how invalidated physical slots are accepted. This patch changes what hot standby feedback sends with interval 0, but it also changes shipped slot behaviour in back branches.

## Objections a reviewer is likely to raise

### 1. Back-patch makes a minor release invalidate logical slots on standbys

A user on 16 to 19 runs hot_standby_feedback = on and wal_receiver_status_interval = 0, with a logical slot or synced failover slot on the standby. After a minor update the primary drops catalog_xmin and catalog VACUUM removes rows. The standby logs "invalidating obsolete replication slot", decoding fails, and the subscriber must be rebuilt.

Message #4, case 5, shows this invalidation with the patch and not without it. Today the stale catalog_xmin protects the slot, even if only as a side effect of the bug. The patch is marked Backpatch-through: 16, but it adds no release note, no warning, and no slotsync.c check that rejects interval 0. A committer would likely be wary of a minor release that breaks replication setups which work today, as the recorded concern in [P8] about changing long-released slot behaviour shows.

Precedent: [Forbid the use of invalidated physical slots in streaming replication.](https://www.postgresql.org/message-id/flat/OS0PR01MB5716A626A4AF5814E057CEE39484A@OS0PR01MB5716.jpnprd01.prod.outlook.com) (Returned with feedback, 2024-02-01)

**Question:** How will users with standby logical or synced slots and interval 0 be protected or warned before a minor update invalidates those slots?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com); `replication-ondisk-1`</sub>

### 2. Interval 0 gets a third meaning, silently overriding hot_standby_feedback

A user upgrading from 15 keeps hot_standby_feedback = on and wal_receiver_status_interval = 0, which sent feedback on every call there. On 16+ with the patch, the standby sends no xmin and logs nothing. The user finds out from query cancellations on the standby or from slotsync running without catalog_xmin protection.

Message #1 shows that 14 and 15 sent feedback on every call with interval 0, so the patch fixes a third behaviour across branches. The proposal does not say why "feedback off" was preferred over restoring the 15 behaviour, which message #4 asked about. The hot_standby_feedback entry in config.sgml says only that feedback is not sent more often than once per interval, and the patch leaves the docs unchanged. Slotsync still accepts this combination without a check or a warning.

**Question:** Why should interval 0 disable feedback that the user turned on explicitly, instead of restoring the 15 behaviour, and where is the user told this happens?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com); `standards-compat-2`</sub>

### 3. Rule that interval 0 disables feedback is not enforced in one place

Slot synchronization checks only hot_standby_feedback, so a standby with interval 0 passes the check while the primary gets no catalog_xmin for synced slots. A later patch that reads hot_standby_feedback alone repeats the bug, and no test fails until a user loses slot protection.

The bug came from two copies of the "feedback enabled" test that disagreed. The patch adds a third copy in walreceiver.c and leaves slotsync.c and config.sgml unaware of the new rule (message #4, comments 2 to 4). Nothing fails loudly when the next caller checks only hot_standby_feedback. A single helper used by walreceiver and slotsync, plus a doc sentence, would close this gap.

**Question:** Will the patch define one "hot standby feedback is enabled" test that walreceiver and slotsync both use, and state the rule in config.sgml?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Why should interval 0 turn hot standby feedback off in 16 to 19, instead of restoring the 15 behaviour of sending feedback on every call?
2. What protects or warns users whose standby logical slots or synced slots are invalidated after the minor update, as message #4 case 5 shows?
3. Will the patch change config.sgml to say that wal_receiver_status_interval = 0 disables hot standby feedback even when hot_standby_feedback is on?
4. Should slot synchronization reject or warn about wal_receiver_status_interval = 0, since the primary then gets no catalog_xmin for synced slots?
5. Is a release note planned for the back branches, and should master and the back branches get the same behaviour?

## Touch points

- **WAL / replication / standby:** The patch changes the xmin that the walreceiver sends in hot standby feedback, which controls slot xmin and catalog_xmin on the primary.
- **new GUC:** No new GUC, but wal_receiver_status_interval = 0 gains the meaning "hot_standby_feedback is off".
- **backpatchability:** The patches are marked Backpatch-through: 16 and change what interval 0 does in a minor release.
- **maintenance burden ("who maintains this in 10 years"):** The "feedback enabled" test now exists in three places in walreceiver.c, and slotsync checks only hot_standby_feedback.

## Who would commit it, and when

- Owner: silent. Someone who knows the walreceiver feedback code, replication slot xmin handling and slot synchronization would have to own it. The visible thread shows the author and one reviewer, and no committer has spoken.
- Priority: fix design first; audience niche; review cost low. Only standbys with hot_standby_feedback = on and wal_receiver_status_interval = 0 are affected, and the patch is one condition in walreceiver.c plus a TAP test. The bug holds back VACUUM on the primary, so it deserves a fix once the interval-0 behaviour and the back-patch effects are agreed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
