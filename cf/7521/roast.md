# Design pre-review: Fix recovery conflict canceling the startup progress timeout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7521](https://commitfest.postgresql.org/patch/7521/); [thread](https://www.postgresql.org/message-id/flat/1D46953B-2FE9-4611-9B46-FAE91184AE99%40gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated, reproduced on 15 through master by the reviewer, and the fix restores behaviour that log_startup_progress_interval already promises. The one surviving objection, prior-attempts-1, is a fixable design point. It asks how the lock and buffer-pin waits behave now that the progress timeout stays armed and can wake the startup process. A committer who knows the recovery-conflict code can settle it, and the backpatch to 15 needs the same person.

Not checked: Without the code, I could not check whether the resolvers re-arm STANDBY_DEADLOCK_TIMEOUT on each wakeup, whether the outdated comment is updated, or whether other startup-process timeouts relied on the old cleanup.

## Prior attempts

- [Revert "Get rid of the dedicated latch for signaling the startup process".](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com) (related; Reverted, 2020-12-17). "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()." Different now: [P3] removed the startup process's dedicated latch. This patch changes which timeouts the resolvers cancel, but it leaves the progress timeout armed as a periodic wakeup source during the same buffer-pin wait.

## Objections a reviewer is likely to raise

### 1. Progress timeout stays armed during conflict waits, which wake the startup process

An admin sets log_startup_progress_interval below deadlock_timeout during PITR with hot_standby=on. Each progress tick sets the latch, so the startup process wakes from its buffer-pin or lock wait and calls the resolver again. If the resolver re-arms STANDBY_DEADLOCK_TIMEOUT on each call, the deadlock check may never fire, and the wait lasts until max_standby_archive_delay, or forever at -1.

[P3] was reverted because it broke the assumption that a startup process waiting on a buffer-pin conflict is woken only by unpin or by the timeout that ResolveRecoveryConflictWithBufferPin() enabled. With this patch, STARTUP_PROGRESS_TIMEOUT stays armed through every conflict wait, which is the kind of extra wakeup [P3] warns about. Today this happens only during the first conflict; the patch makes it happen in all of them. Messages #1-#3 do not discuss wakeups during the wait.

Precedent: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com) (Reverted, 2020-12-17)

**Question:** How do the lock and buffer-pin waits behave when the progress timeout fires repeatedly, for example with log_startup_progress_interval shorter than deadlock_timeout?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/1D46953B-2FE9-4611-9B46-FAE91184AE99%40gmail.com), [#3](https://www.postgresql.org/message-id/z-F0YA2lTC4VlSp9RRxzwY9HDXFFyfxWoeWd7y_ITHcYiK6BWW0kgwMFTv18HL82L1_sy0kDr91fJwtKmkTJ1rCx6OMI6niU73yIuFLej2I%3D%40rtshkmr.com); `prior-attempts-1`</sub>

## Questions for the author

1. How do the lock and buffer-pin waits behave when the progress timeout fires repeatedly, for example with log_startup_progress_interval shorter than deadlock_timeout?
2. Does any other startup-process code arm a timeout and rely on the old disable_all_timeouts() call to clear it?
3. Does the patch update or remove the comment that says the startup process has no other outstanding timeouts?
4. If CF 7372 is committed first, which entry carries the narrower cleanup into its reworked resolvers?
5. Why did the entry move from Ready for Committer back to Needs review on 2026-11-03?

## Touch points

- **WAL / replication / standby:** The patch changes timeout cleanup in the startup process's lock and buffer-pin recovery-conflict resolvers in standby.c.
- **backpatchability:** The thread proposes backpatching the fix to 15 and the test to 17, with per-branch test variants.
- **maintenance burden ("who maintains this in 10 years"):** CF 7372 reworks the same resolvers and must keep the narrower cleanup; the test needs different variants on back branches.

## Who would commit it, and when

- Owner: silent. Someone who knows the startup process's recovery-conflict waits and the timeout and latch code in standby.c would have to own it.
- Priority: proceed; audience niche; review cost low. The fix touches one file and narrows two cleanup calls, and the reviewer verified it on 15 through master with a test. It affects admins who run queries during archive recovery with hot_standby=on, and it needs one answer on wakeups during the wait.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
