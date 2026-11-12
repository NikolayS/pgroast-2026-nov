# Design pre-review: Persist slot invalidations before publishing them

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7203](https://commitfest.postgresql.org/patch/7203/); [thread](https://www.postgresql.org/message-id/flat/ao7u5I9OeIR72kGp%40bdtpg), 42 of 42 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the master design; three reviewers accept the approach through v11. The one surviving objection, maintenance-fit-1, is fixable: 0001 changes who holds io_in_progress_lock and is proposed for backpatch to 14, but versions for 14 to 18 do not exist. A committer who knows the slot code has to agree to the locking change and the backpatch range.

Not checked: The code was not read, so the lock ordering, the error-path cleanup and the stability of the injection-point tests are unchecked. The backpatch could not be judged because versions for 14 to 18 do not exist.

## Prior attempts

- [Revert "Add recovery TAP test for race condition with slot invalidations"](https://www.postgresql.org/message-id/flat/ZekQQHCrIqLVpGz5@paquier.xyz) (related; Reverted, 2024-03-07). "Getting the test right should require some redesign with a second injection point, but let's revert it for now to avoid these issues in the CI as a lot of patches are under discussion in this last commit fest." Different now: P3 reverted an unstable race-condition TAP test for slot invalidation. This proposal is a durability fix, but it adds an injection-point test of two concurrent invalidators, so a similar test-stability risk applies.
- [Revert "Add test case for obsoleting slot with active walsender"](https://www.postgresql.org/message-id/flat/202106102202.mjw4huiix7lo@alvherre.pgsql) (related; Reverted, 2021-06-20). "the test case added there failed once in circumstances that remain mysterious." Different now: P8 reverted a test that made a slot obsolete while a walsender was active, after an unexplained failure. This proposal does not repeat that test, but it adds new slot invalidation tests in the same test suite.

## Objections a reviewer is likely to raise

### 1. Backpatch of a new slot locking protocol to 14 without branch versions

A user on 14 or 15 installs a minor release that carries the adapted 0001. The checkpointer now takes each slot's io_in_progress_lock during invalidation, and an unreviewed back-branch difference in the slot code makes it wait or deadlock. The user sees checkpoints stall after a routine minor upgrade.

0001 changes who holds io_in_progress_lock and for how long, and it adds a wait-and-restart loop that drops ReplicationSlotControlLock (message #10). That protocol was reviewed on master only, and versions for 14 to 18 do not exist (messages #2, #3). Message #1 says 14 and 15 need adaptation that nobody has examined. A committer would likely want to see each branch version before agreeing to Backpatch-through: 14, since a locking change in a minor release is hard to take back.

**Question:** What do the 14 and 15 versions of 0001 look like, and does the lock-ordering argument from message #10 hold unchanged on each branch?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/ao7u5I9OeIR72kGp%40bdtpg), [#2](https://www.postgresql.org/message-id/c1xYEcyG6PvNaeTQlBD5VAsH9K1-41RTiB4QdODvpkhvAGANt5S_cPD-GsIN7vLzKvBrMt1p-PD0CelhGl_zqgwLZnTTKPunf3xYiVKWE0U%3D%40proton.me), [#3](https://www.postgresql.org/message-id/ao%2BTI4WNe0gKpiP3%40bdtpg), [#10](https://www.postgresql.org/message-id/CAHWVJhF33kdDOE9VVRUM24BD8ALTX5ayuK8roDR8EtpHkmnX7g%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What do the 14 and 15 versions of 0001 look like, and does the lock-ordering argument from message #10 hold unchanged on each branch?
2. Should the patch close the window in synchronize_one_slot() where a restartpoint can terminate the slot sync worker between acquiring the slot and taking the lock?
3. Is it acceptable that a failed save deletes a temporary synchronized slot?
4. On a hot standby, what stops a connection from using a restored valid slot before the next restartpoint detects the invalidation again?

## Touch points

- **on-disk format / pg_upgrade:** The patch changes when the slot state file is written and fsynced; the file format stays the same.
- **WAL / replication / standby:** The patch changes replication slot invalidation in slot.c and slot synchronization in slotsync.c.
- **ABI / extension API:** The patch adds ReplicationSlotPersistInvalidation() to replication/slot.h, also in back branches.
- **backpatchability:** 0001 targets 14 and 0002 targets 17, and no branch versions exist for 14 to 18.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a locking protocol on io_in_progress_lock for invalidators and two injection-point TAP tests.
- **performance on the hot path:** The checkpoint pass over 100 valid physical slots went from 7.2 us to 9.0 us.

## Who would commit it, and when

- Owner: silent. Someone who knows replication slot locking, checkpointer slot invalidation and slot synchronization. The brief says no committer has stated a position on the design or the backpatch.
- Priority: needs champion; audience a sizeable minority; review cost medium. The bug can restore a valid slot after its WAL or rows were removed, so it matters to users of replication slots. Review on master has converged; the entry waits on a committer and on branch versions for 14 to 18.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
