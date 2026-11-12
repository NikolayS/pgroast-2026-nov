# Design pre-review: Persist slot invalidations before publishing them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7203](https://commitfest.postgresql.org/patch/7203/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Persist slot invalidations before publishing them](https://www.postgresql.org/message-id/flat/ao7u5I9OeIR72kGp%40bdtpg), started 2026-08-26
- Messages read: 42 of 42
- Patch prose read: v11-0001-Persist-slot-invalidations-before-publishing-the.patch, v11-0002-Persist-synchronized-slot-invalidations-before-p.patch, v1-0001-Persist-slot-invalidations-before-publishing-them.patch, v1-0002-Persist-synchronized-slot-invalidations-before-pu.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the master design; three reviewers accept the approach through v11. The one surviving objection, maintenance-fit-1, is fixable: 0001 changes who holds io_in_progress_lock and is proposed for backpatch to 14, but versions for 14 to 18 do not exist. A committer who knows the slot code has to agree to the locking change and the backpatch range.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the lock ordering, the error-path cleanup and the stability of the injection-point tests are unchecked. The backpatch could not be judged because versions for 14 to 18 do not exist.

## The proposal, restated

**Problem.** An inactive replication slot is marked invalid in shared memory before its state file is saved. If the save fails or the server crashes first, horizon computations may already have removed WAL or rows, and restart restores the old valid slot. In synchronize_one_slot() a failed save also blocks a direct retry, because the next sync sees the local slot as already invalid.

**Proposed design.** A new ReplicationSlotPersistInvalidation() writes and fsyncs an invalidated copy of the acquired slot while the shared slot stays valid, then publishes the invalidation. The slot's io_in_progress_lock is held until publication, and it also serializes concurrent internal invalidators so the second one waits instead of signalling the first. The fix covers InvalidatePossiblyObsoleteSlot() (0001) and synchronize_one_slot() (0002).

**User-visible surface.**
- C API: new ReplicationSlotPersistInvalidation() declared in replication/slot.h.
- Wait event ReplicationSlotIO can now appear for a second invalidator waiting on the same slot.
- New injection point in the slot invalidation path, used by the tests.

**What the author claims.**
- The on-disk slot format and the ReplicationSlot shared memory layout do not change, which eases backpatching.
- A failed save leaves both the shared slot and its disk image valid, and a later checkpoint or sync retries without a new retry mechanism.
- 0001 should be backpatched to 14 and 0002 to 17; versions 14 and 15 need adaptation.
- Rui Zhao measured a checkpoint pass over 100 valid physical slots going from 7.2 us to 9.0 us with the locking change.
- On a hot standby, connections can use a restored valid slot before the next restartpoint detects the invalidation again.

**Thread state.** Rui Zhao, shveta malik and Ashutosh Sharma reviewed through v11 and say the approach and code look good, after the locking fix for concurrent invalidators was folded in. Horiguchi proposed a separate invalidation file and Zhijie Hou an in-memory durability flag; the author argued against both, and the latest messages are rebases.

## 2. Prior attempts

### Revert "Add recovery TAP test for race condition with slot invalidations" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643)
- **Thread:** [https://www.postgresql.org/message-id/flat/ZekQQHCrIqLVpGz5@paquier.xyz](https://www.postgresql.org/message-id/flat/ZekQQHCrIqLVpGz5@paquier.xyz), [https://www.postgresql.org/message-id/flat/ZdyZya4YrNapWKqz@ip-10-97-1-34.eu-west-3.compute.internal](https://www.postgresql.org/message-id/flat/ZdyZya4YrNapWKqz@ip-10-97-1-34.eu-west-3.compute.internal)
- **Date:** decided 2024-03-07; started 2024-03-06
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 08a52ab151ca, due to some sporadic instability in the test. Getting the test right should require some redesign with a second injection point, but let's revert it for now to avoid these issues in the CI as a lot of patches are under discussion in this last commit fest. Per buildfarm members hachi and gokiburi.
- **Reason (quote):** "Getting the test right should require some redesign with a second injection point, but let's revert it for now to avoid these issues in the CI as a lot of patches are under discussion in this last commit fest."
- **What is different this time:** P3 reverted an unstable race-condition TAP test for slot invalidation. This proposal is a durability fix, but it adds an injection-point test of two concurrent invalidators, so a similar test-stability risk applies.

### Revert "Add test case for obsoleting slot with active walsender" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95)
- **Thread:** [https://www.postgresql.org/message-id/flat/202106102202.mjw4huiix7lo@alvherre.pgsql](https://www.postgresql.org/message-id/flat/202106102202.mjw4huiix7lo@alvherre.pgsql)
- **Date:** decided 2021-06-20; started 2021-06-18
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 09126984a263; the test case added there failed once in circumstances that remain mysterious. It seems better to remove the test for now so that 14beta2 doesn't have random failures built in.
- **Reason (quote):** "the test case added there failed once in circumstances that remain mysterious."
- **What is different this time:** P8 reverted a test that made a slot obsolete while a walsender was active, after an unexplained failure. This proposal does not repeat that test, but it adds new slot invalidation tests in the same test suite.


## 3. Objections a reviewer is likely to raise

### 1. Backpatch of a new slot locking protocol to 14 without branch versions

fixable · level 2, rubric item 6 · backpatchability, maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A user on 14 or 15 installs a minor release that carries the adapted 0001. The checkpointer now takes each slot's io_in_progress_lock during invalidation, and an unreviewed back-branch difference in the slot code makes it wait or deadlock. The user sees checkpoints stall after a routine minor upgrade.

0001 changes who holds io_in_progress_lock and for how long, and it adds a wait-and-restart loop that drops ReplicationSlotControlLock (message #10). That protocol was reviewed on master only, and versions for 14 to 18 do not exist (messages #2, #3). Message #1 says 14 and 15 need adaptation that nobody has examined. A committer would likely want to see each branch version before agreeing to Backpatch-through: 14, since a locking change in a minor release is hard to take back.

Thread: [#1 (Bertrand Drouvot, 2026-08-26)](https://www.postgresql.org/message-id/ao7u5I9OeIR72kGp%40bdtpg), [#2 (Miłosz Bieniek, 2026-08-26)](https://www.postgresql.org/message-id/c1xYEcyG6PvNaeTQlBD5VAsH9K1-41RTiB4QdODvpkhvAGANt5S_cPD-GsIN7vLzKvBrMt1p-PD0CelhGl_zqgwLZnTTKPunf3xYiVKWE0U%3D%40proton.me), [#3 (Bertrand Drouvot, 2026-08-27)](https://www.postgresql.org/message-id/ao%2BTI4WNe0gKpiP3%40bdtpg), [#10 (Rui Zhao, 2026-09-17)](https://www.postgresql.org/message-id/CAHWVJhF33kdDOE9VVRUM24BD8ALTX5ayuK8roDR8EtpHkmnX7g%40mail.gmail.com)

**Question:** What do the 14 and 15 versions of 0001 look like, and does the lock-ordering argument from message #10 hold unchanged on each branch?

Counter-check: held. Message #3 defers the branch versions until the master fix is agreed, which is usual practice but does not answer the request. Message #1 says 14 and 15 need adaptation, and the lock-ordering argument in message #10 was checked on master only.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch changes when the slot state file is written and fsynced; the file format stays the same. |
| WAL / replication / standby | **yes** | The patch changes replication slot invalidation in slot.c and slot synchronization in slotsync.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds ReplicationSlotPersistInvalidation() to replication/slot.h, also in back branches. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | 0001 targets 14 and 0002 targets 17, and no branch versions exist for 14 to 18. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a locking protocol on io_in_progress_lock for invalidators and two injection-point TAP tests. |
| performance on the hot path | **yes** | The checkpoint pass over 100 valid physical slots went from 7.2 us to 9.0 us. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What do the 14 and 15 versions of 0001 look like, and does the lock-ordering argument from message #10 hold unchanged on each branch?
2. Should the patch close the window in synchronize_one_slot() where a restartpoint can terminate the slot sync worker between acquiring the slot and taking the lock?
3. Is it acceptable that a failed save deletes a temporary synchronized slot?
4. On a hot standby, what stops a connection from using a restored valid slot before the next restartpoint detects the invalidation again?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows replication slot locking, checkpointer slot invalidation and slot synchronization. The brief says no committer has stated a position on the design or the backpatch.
- **Liability:** The owner signs for a changed io_in_progress_lock protocol in the checkpointer path on every supported branch back to 14, for the life of those branches, plus the stability of two injection-point tests.
- **Author follow-up:** The author answered reviews, folded in the fix for concurrent invalidators and kept rebasing through v11. Message #3 promises branch versions once the master fix is agreed.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug can restore a valid slot after its WAL or rows were removed, so it matters to users of replication slots. Review on master has converged; the entry waits on a committer and on branch versions for 14 to 18.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` New lock-ordering rule exists only in the thread. Refuted (already handled): Messages #31, #38 and #39 show assertions on I/O lock ownership at each layer, so a caller that breaks the contract fails in assert builds. Postgres enforces no LWLock ordering rule at run time, and where the comment sits is a code-level question.

### Review angles

- prior-attempts: 0 objection(s). No precedent rejected the idea of persisting a slot invalidation before publishing it. P3 and P8 were test reverts, which are an implementation-level concern outside this review, and P2 rejected a different change to physical slot behaviour. Message #1 cites the commits that introduced the current ordering, not earlier attempts to change it.
- standards-compat: 0 objection(s). The proposal changes no SQL syntax, GUC, catalog, dumped object, psql behaviour or wire message, and it keeps the on-disk slot format and the ReplicationSlot layout, so back branches see no contract change. The only client-visible effects are a ReplicationSlotIO wait in a rare race and a slot that stays valid after a failed save, which fixes the current state. The 14 and 15 backpatch versions do not exist yet, so their compatibility could not be checked.
- replication-ondisk: 0 objection(s). No objection from the replication and on-disk angle. The proposal keeps the slot state file format and the ReplicationSlot shared memory layout, adds no WAL record, catalog or protocol change, and leaves pg_upgrade untouched. It moves only the ordering, so an invalid image reaches disk before horizons stop counting the slot. The new disk-ahead-of-memory window fails safe: a crash there restores an invalid slot, and the invalidation was already decided. Thread messages #10 and #23 found and settled the concurrent-invalidator termination and the temporary synchronized slot case. The slot sync worker window in message #10 (d) also exists on master. The 14 and 15 backport versions do not exist yet, so the backport cannot be checked.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It reorders a slot state write before publication inside slot.c and slotsync.c, adds no SQL interface, role, GUC or view column, and changes nothing a non-superuser can do or see. The new io_in_progress_lock wait happens only between internal invalidators, so an unprivileged session cannot hold it to block them.
- maintenance-fit: 2 objection(s). The fix is a bugfix in a narrow path. The hot-path cost is small and shown (message #10), and it adds no new dependency. Remaining concerns are backpatch scope and keeping the locking rule explicit.
- already-exists: 0 objection(s). This is a crash-durability bug fix inside slot.c and slotsync.c, and no existing feature, setting or extension can change the order in which core saves and publishes a slot invalidation. The patch also reuses existing mechanisms, the slot state file and io_in_progress_lock, so it adds no second mechanism for the same job.
- demand: 0 objection(s). This is a durability bugfix, so demand does not need to come from field reports. The thread shows a concrete failure: a failed save or crash leaves a valid slot on disk after its WAL or rows were removed, and on master a failed save leaves the slot stuck active (message #10). Several independent reviewers agree the problem is real. The cost falls on the invalidation path and a small checkpoint pass overhead, so a demand-based objection would not hold.

### Precedents retrieved

- P1: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P2: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P3: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - used in the review
- P4: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P5: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P6: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P7: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P8: [Revert "Add test case for obsoleting slot with active walsender"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95) (revert, Reverted, 2021-06-20) - used in the review
- P9: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P10: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P11: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P12: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P14: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant
- P15: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P16: [Problem in 'ORDER BY' of a column using a created collation?](https://commitfest.postgresql.org/patch/5990/) (cf, Returned with feedback, 2025-10-21) - judged not relevant
- P17: [logical column ordering (WIP)](https://commitfest.postgresql.org/patch/21/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P18: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P19: [Make the optimiser aware of partitions ordering](https://commitfest.postgresql.org/patch/1093/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P20: [Optimize SnapBuild by maintaining committed.xip in sorted order](https://commitfest.postgresql.org/patch/6187/) (cf, Withdrawn, 2026-04-29) - judged not relevant
- P21: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P22: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P23: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P24: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 115 s; cost reported by the CLI: $2.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 57 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:28:58.916Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
