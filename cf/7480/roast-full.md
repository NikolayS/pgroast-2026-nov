# Design pre-review: walreceiver: stale hot_standby_feedback xmin on primary when wal_receiver_status_interval = 0 (PG16 and later)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7480](https://commitfest.postgresql.org/patch/7480/) (Needs review); authors: Vaibhav Dalvi (vaibhavdalvi)
- Thread: [walreceiver: stale hot_standby_feedback xmin on primary when wal_receiver_status_interval = 0 (PG16 and later)](https://www.postgresql.org/message-id/flat/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), started 2026-10-15
- Messages read: 4 of 4
- Patch prose read: v2-walreceiver-Do-not-leave-stale-xmin-on-primary-when-status-interval-is-zero-Master.patch, v1-walreceiver-Do-not-leave-stale-xmin-on-primary-when-status-interval-is-zero-PG18.patch, v1-walreceiver-Do-not-leave-stale-xmin-on-primary-when-status-interval-is-zero-Master.patch, v1-walreceiver-Do-not-leave-stale-xmin-on-primary-when-status-interval-is-zero-PG16-17.patch, v1-walreceiver-Do-not-leave-stale-xmin-on-primary-when-status-interval-is-zero-PG19.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The bug is reproduced and the goal is not in dispute. The open point is which interval-0 behaviour back branches should get: replication-ondisk-1 shows that the patched build invalidates logical slots on a standby that work today, and standards-compat-2 shows the choice of "feedback off" over the 15 behaviour is not argued. maintenance-fit-2 adds that slotsync still accepts interval 0. The list should agree on the semantics before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code and the TAP test were not read, so the stability of the test and the exact effect on primary_has_standby_xmin could not be checked. Message #4 is known only through the brief and the objections.

## The proposal, restated

**Problem.** With hot_standby_feedback = on and wal_receiver_status_interval = 0, Postgres 16 and later send one feedback message with a valid xmin at connect or reload, then nothing more. The primary keeps that xmin until the next reconnect, and VACUUM on the primary cannot remove dead rows. Postgres 14 and 15 are not affected.

**Proposed design.** The walreceiver computes an xmin only when hot_standby_feedback is on and wal_receiver_status_interval > 0, so with interval 0 it sends InvalidTransactionId and the primary releases the xmin. This makes interval 0 mean "no hot standby feedback", which matches the existing early return and wakeup logic. A new TAP test in src/test/recovery checks the slot xmin and is meant for back-patching to 16.

**User-visible surface.**
- Changed behaviour: hot_standby_feedback = on with wal_receiver_status_interval = 0 now sends no xmin, so the primary holds no xmin or catalog_xmin for the standby.

**What the author claims.**
- The bug affects 16, 17, 18, 19 and master, and the same code change applies to all of them.
- The fix keeps the documented meaning that 0 disables status updates.
- The new TAP test fails without the fix and passes with it, without timing races.
- Postgres 14 and 15 keep advancing the xmin with interval 0, because their once-per-interval check is always true.

**Thread state.** Joao Rito reviewed v2 on master, confirmed the bug and the fix, and asked which interval-0 behaviour is intended, since 15 kept sending feedback and the patch sends none. He also reported that logical slots on a standby get invalidated with the fix, and asked for doc changes and possibly a warning or a slotsync check.

## 2. Prior attempts

### Forbid the use of invalidated physical slots in streaming replication. (related)

- **Link:** [https://commitfest.postgresql.org/patch/4686/](https://commitfest.postgresql.org/patch/4686/)
- **Thread:** [https://www.postgresql.org/message-id/flat/OS0PR01MB5716A626A4AF5814E057CEE39484A@OS0PR01MB5716.jpnprd01.prod.outlook.com](https://www.postgresql.org/message-id/flat/OS0PR01MB5716A626A4AF5814E057CEE39484A@OS0PR01MB5716.jpnprd01.prod.outlook.com)
- **Date:** decided 2024-02-01; started 2023-12-06
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Ashutosh Bapat, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped responding to review questions, so vignesh C moved the entry to Waiting on Author and then to Returned with Feedback. Ashutosh Bapat asked how a build before commit 15f8203 behaves and warned that changing long-released behaviour of physical slots could break backward compatibility. Robert Haas noted that the documentation does not match the current behaviour and that the intended interaction with physical slots is not clear.
- **Reason (quote):** "I think making logical and physical slot behaviour consistent would be better but if the inconsistent behaviour is out there for some releases, changing that now will break backward compatibility." ([message](https://www.postgresql.org/message-id/CAExHW5t9w_iSU%2BVP-GqGLeRErRJnGzkqavxzYfCk8Z%2Be0pu-3w%40mail.gmail.com))
- **What is different this time:** P8 changed how invalidated physical slots are accepted. This patch changes what hot standby feedback sends with interval 0, but it also changes shipped slot behaviour in back branches.


## 3. Objections a reviewer is likely to raise

### 1. Back-patch makes a minor release invalidate logical slots on standbys

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `replication-ondisk-1`

A user on 16 to 19 runs hot_standby_feedback = on and wal_receiver_status_interval = 0, with a logical slot or synced failover slot on the standby. After a minor update the primary drops catalog_xmin and catalog VACUUM removes rows. The standby logs "invalidating obsolete replication slot", decoding fails, and the subscriber must be rebuilt.

Message #4, case 5, shows this invalidation with the patch and not without it. Today the stale catalog_xmin protects the slot, even if only as a side effect of the bug. The patch is marked Backpatch-through: 16, but it adds no release note, no warning, and no slotsync.c check that rejects interval 0. A committer would likely be wary of a minor release that breaks replication setups which work today, as the recorded concern in [P8] about changing long-released slot behaviour shows.

**Precedents.**
- [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (Returned with feedback, 2024-02-01); thread: [link](https://www.postgresql.org/message-id/flat/OS0PR01MB5716A626A4AF5814E057CEE39484A@OS0PR01MB5716.jpnprd01.prod.outlook.com)

Thread: [#1 (Vaibhav Dalvi, 2026-10-15)](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com)

**Question:** How will users with standby logical or synced slots and interval 0 be protected or warned before a minor update invalidates those slots?

Counter-check: held. Message #4, case 5, shows the invalidation on the patched build only, and the patch is marked Backpatch-through: 16 with no release note or warning. [P8] is weak support, because that entry was returned after its author stopped replying.

### 2. Interval 0 gets a third meaning, silently overriding hot_standby_feedback

fixable · level 2, rubric item 12 · new GUC, backpatchability · `standards-compat-2`

A user upgrading from 15 keeps hot_standby_feedback = on and wal_receiver_status_interval = 0, which sent feedback on every call there. On 16+ with the patch, the standby sends no xmin and logs nothing. The user finds out from query cancellations on the standby or from slotsync running without catalog_xmin protection.

Message #1 shows that 14 and 15 sent feedback on every call with interval 0, so the patch fixes a third behaviour across branches. The proposal does not say why "feedback off" was preferred over restoring the 15 behaviour, which message #4 asked about. The hot_standby_feedback entry in config.sgml says only that feedback is not sent more often than once per interval, and the patch leaves the docs unchanged. Slotsync still accepts this combination without a check or a warning.

Thread: [#1 (Vaibhav Dalvi, 2026-10-15)](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com)

**Question:** Why should interval 0 disable feedback that the user turned on explicitly, instead of restoring the 15 behaviour, and where is the user told this happens?

Counter-check: held. The existing docs say interval 0 disables status updates, which supports the author's choice. But message #4 asked why the 15 behaviour was not restored and the thread has no answer, and the patch does not change config.sgml.

### 3. Rule that interval 0 disables feedback is not enforced in one place

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-2`

Slot synchronization checks only hot_standby_feedback, so a standby with interval 0 passes the check while the primary gets no catalog_xmin for synced slots. A later patch that reads hot_standby_feedback alone repeats the bug, and no test fails until a user loses slot protection.

The bug came from two copies of the "feedback enabled" test that disagreed. The patch adds a third copy in walreceiver.c and leaves slotsync.c and config.sgml unaware of the new rule (message #4, comments 2 to 4). Nothing fails loudly when the next caller checks only hot_standby_feedback. A single helper used by walreceiver and slotsync, plus a doc sentence, would close this gap.

Thread: [#1 (Vaibhav Dalvi, 2026-10-15)](https://www.postgresql.org/message-id/CA%2BvB%3DAHvkSsYXNNiVKakQtOEdyQBv7kob%3DLGESOdOC7AC0M9Ow%40mail.gmail.com), [#4 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110054328.44164-1-joaovictor.rito%40gmail.com)

**Question:** Will the patch define one "hot standby feedback is enabled" test that walreceiver and slotsync both use, and state the rule in config.sgml?

Counter-check: held. The helper in walreceiver.c is a code structure point, which message #4 calls a matter of taste. The slotsync check is a design gap: with the patch it accepts interval 0 while the primary gets no catalog_xmin for synced slots.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes the xmin that the walreceiver sends in hot standby feedback, which controls slot xmin and catalog_xmin on the primary. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but wal_receiver_status_interval = 0 gains the meaning "hot_standby_feedback is off". |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The patches are marked Backpatch-through: 16 and change what interval 0 does in a minor release. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The "feedback enabled" test now exists in three places in walreceiver.c, and slotsync checks only hot_standby_feedback. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should interval 0 turn hot standby feedback off in 16 to 19, instead of restoring the 15 behaviour of sending feedback on every call?
2. What protects or warns users whose standby logical slots or synced slots are invalidated after the minor update, as message #4 case 5 shows?
3. Will the patch change config.sgml to say that wal_receiver_status_interval = 0 disables hot standby feedback even when hot_standby_feedback is on?
4. Should slot synchronization reject or warn about wal_receiver_status_interval = 0, since the primary then gets no catalog_xmin for synced slots?
5. Is a release note planned for the back branches, and should master and the back branches get the same behaviour?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the walreceiver feedback code, replication slot xmin handling and slot synchronization would have to own it. The visible thread shows the author and one reviewer, and no committer has spoken.
- **Liability:** The owner signs for a behaviour change in four back branches: any report of invalidated standby logical slots or cancelled standby queries after the minor update. The owner also signs for the interval-0 semantics in all later releases.
- **Author follow-up:** The author rebased within four days and said a separate thread will cover the related stale-xmin case. The visible thread has no reply yet to the review in message #4.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only standbys with hot_standby_feedback = on and wal_receiver_status_interval = 0 are affected, and the patch is one condition in walreceiver.c plus a TAP test. The bug holds back VACUUM on the primary, so it deserves a fix once the interval-0 behaviour and the back-patch effects are agreed.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Back-patched change to interval 0 can invalidate standby logical slots. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same slot invalidation and names the missing release note, warning and slotsync check. The question about the 15 behaviour is covered by standards-compat-2.
- `standards-compat-1` Back-patched fix removes xmin protection that existing standbys rely on. Refuted (not specific): Duplicate of replication-ondisk-1. The scenario and the evidence from message #4, case 5, are the same.
- `replication-ondisk-2` Choice of "no feedback" over restoring the 15 behaviour is not argued. Refuted (not specific): Duplicate of standards-compat-2, which makes the same point about the unanswered 15-behaviour question and the unchanged config.sgml.
- `maintenance-fit-1` Back-patching "interval 0 means feedback off" breaks standby logical slots. Refuted (not specific): Duplicate of replication-ondisk-1 for the slot invalidation and of standards-compat-2 for the choice between the two fixes.
- `already-exists-1` Interval 0 becomes a second off-switch for hot standby feedback. Refuted (not specific): Duplicate of standards-compat-2. The point that interval 0 overrides an explicit hot_standby_feedback = on without a doc change is the same.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes the same fix. P4 is about feedback under heavy load, P3 and P5 are about the feedback epoch, and P23 is an unstable vacuum test. None of their recorded reasons applies here.
- standards-compat: 2 objection(s). The bug itself is not in dispute. Both objections concern the chosen semantics and how they reach back branches.
- replication-ondisk: 2 objection(s). The patch changes no on-disk format, WAL record, catalog, wire message or ABI. Only the xmin value in an existing feedback message changes.
- security: 0 objection(s). The change only affects which xmin the walreceiver sends when hot_standby_feedback is on and wal_receiver_status_interval is 0. Both settings are set by the standby administrator, so no new privilege, exposure or path for an unprivileged user exists, and the fix reduces the bloat risk on the primary.
- maintenance-fit: 2 objection(s). There is no hot-path cost, no new dependency and no fit issue. This is a small bugfix in core code.
- already-exists: 1 objection(s). The bug fix itself is not covered by any existing feature or extension. Setting a positive interval avoids the bug but does not fix it.
- demand: 0 objection(s). This is a bug fix, so the demand test applies weakly: the author and the reviewer in message #4 both reproduced vacuum being held back on 16 and later. The cost falls only on standbys set to wal_receiver_status_interval = 0, and the choice between the two interval-0 behaviours belongs to the design angle.

### Precedents retrieved

- P1: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P2: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P3: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - used in the review; reason summarised from its thread
- P4: [Sending WAL receiver feedback regularly even if the receiver is under heavy load](https://commitfest.postgresql.org/patch/143/) (cf, Returned with feedback, 2015-05-16) - used in the review; reason summarised from its thread
- P5: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - used in the review; reason summarised from its thread
- P6: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P7: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P8: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P9: [Fix possible segfault in walreceiver](https://commitfest.postgresql.org/patch/2873/) (cf, Withdrawn, 2020-12-17) - judged not relevant; reason summarised from its thread
- P10: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P11: [suppressing useless wakeups in logical/worker.c](https://commitfest.postgresql.org/patch/4035/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P12: [low wal_retrieve_retry_interval causes missed signals on Windows](https://commitfest.postgresql.org/patch/4118/) (cf, Withdrawn, 2023-02-25) - judged not relevant
- P13: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - judged not relevant
- P14: [delay starting WAL receiver](https://commitfest.postgresql.org/patch/4117/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P15: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P16: [Revert "Use WAIT FOR LSN in PostgreSQL::Test::Cluster::wait_for_catchup()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e54ce0b2da62ac71e3b2a5212f32e82b07bf39a5) (revert, Reverted, 2026-01-07) - judged not relevant
- P17: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P18: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P19: [Creating backup history files for backups taken from standbys](https://commitfest.postgresql.org/patch/1231/) (cf, Returned with feedback, 2018-03-05) - judged not relevant
- P20: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P21: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P22: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4750/) (cf, Withdrawn, 2024-10-12) - judged not relevant
- P23: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - used in the review
- P24: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 115 s; cost reported by the CLI: $1.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 85 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:00:09.543Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
