# Design pre-review: REPACK (CONCURRENTLY): do not block the table while waiting for the final lock

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7369](https://commitfest.postgresql.org/patch/7369/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is real and measured, and no surviving objection is fatal. The design leaves polling unbounded when lock_timeout is unset (standards-compat-1), and the decoding slot retains WAL and catalog_xmin for that whole time (replication-ondisk-1). The deadlock fallback covers only waiters on the table lock (maintenance-fit-3). Message #2 offers a competing design, and the thread has not compared the two.

Not checked: The code was not read. Whether the fallback through LockHasWaitersRelation() covers every lock REPACK holds while polling, and how the loop interacts with the decoding worker, cannot be judged from the prose.

## Prior attempts

None found before 2026-09-30.

## Objections a reviewer is likely to raise

### 1. Default REPACK may never finish when lock_timeout is unset

A DBA runs REPACK (CONCURRENTLY) without lock_timeout on a table with steady short-lock traffic. Conditional attempts keep failing, and REPACK keeps decoding and holding its slot for hours. The DBA notices only when it has not finished and WAL has piled up.

Today REPACK without lock_timeout queues and gets the lock once the blocker ends, so it always finishes. The patch changes that contract with no option to opt out. Message #1 measured 1 to 10 s at 135k tps but gives no upper bound, and a busier table may never see a gap. The LockHasWaitersRelation() fallback only covers waiters behind REPACK's own lock.

**Question:** What bounds polling when lock_timeout is unset, and why is there no REPACK option to keep the current queueing behavior?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Unbounded polling keeps the REPACK decoding slot and its WAL alive

A DBA runs REPACK (CONCURRENTLY) on a hot table with lock_timeout unset. Conditional attempts keep failing because some lock is always held, so the logical decoding slot holds back WAL and catalog_xmin while REPACK polls. pg_wal grows until the disk fills, or max_slot_wal_keep_size invalidates the slot and REPACK fails after all the copying.

Message #1 reports 1 to 10 s to get the lock at 135k tps, but gives no upper bound when lock_timeout is unset. Queueing waits only for the holders ahead of it, while polling can lose every attempt to new arrivals. The proposal does not say how long REPACK may keep its slot during polling, or what happens to WAL retention and catalog bloat in that time.

**Question:** What bounds polling when lock_timeout is unset, and how is WAL and catalog_xmin retention by the REPACK slot limited while it polls?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Deadlock fallback depends on a check later changes can break

A later patch makes REPACK hold a lock on another object while it polls. A session waits on that lock while it holds a lock on the table. LockHasWaitersRelation() on the table reports nothing, so REPACK keeps polling and the deadlock detector never runs, and the hang lasts until lock_timeout or manual cancellation.

Message #1 limits the fallback to waiters behind "the lock REPACK holds" on the relation. Any cycle through another lock REPACK holds is outside that check. Correctness then depends on every future change to REPACK's locking also updating the waiter check. A reviewer is likely to miss that, and nothing fails loudly when it is missed.

**Question:** Which locks does REPACK hold while polling, and what ensures that a waiter on any of them, now or after later changes, triggers the normal wait path?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com); `maintenance-fit-3`</sub>

### 4. Lock monitoring no longer shows what blocks REPACK

A DBA sees REPACK stuck and runs pg_blocking_pids() on its pid, or looks for an ungranted row in pg_locks, to find the long transaction. With polling there is no waiting lock request, so both return nothing. The DBA cannot identify the session to cancel.

Lock waits in Postgres show up in pg_locks and pg_blocking_pids(), and existing monitoring scripts depend on that. The patch replaces the lock wait with a new wait event, REPACK_LOCK_WAIT, which names no blocker. The documentation hunk does not tell users how to find the blocking session.

**Question:** How does a user find the session that blocks a polling REPACK, given that pg_locks and pg_blocking_pids() no longer show the wait?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What bounds polling when lock_timeout is unset, and can a steady stream of short AccessShareLock holders starve REPACK forever?
2. How much WAL and catalog_xmin does the REPACK slot retain while polling on a write-heavy table, and what limits it?
3. Which locks does REPACK hold while polling, and what makes a waiter on any of them trigger the normal wait path?
4. How does this patch compare with the patch in message #2, which limits the wait without letting REPACK give up?
5. How does a user find the session that blocks a polling REPACK, now that pg_blocking_pids() returns nothing for it?

## Touch points

- **WAL / replication / standby:** REPACK keeps its logical decoding slot, and so WAL and catalog_xmin, for as long as it polls.
- **catalog changes:** see objection replication-ondisk-1: Unbounded polling keeps the REPACK decoding slot and its WAL alive
- **psql UX conventions:** see objection standards-compat-2: Lock monitoring no longer shows what blocks REPACK
- **backpatchability:** The patch changes REPACK behavior; message #3 refuses that for 19, so it targets 20.
- **maintenance burden ("who maintains this in 10 years"):** The polling loop in repack.c has its own deadlock fallback, and later locking changes in REPACK must keep it correct.

## Who would commit it, and when

- Owner: qualified committer engaged. Alvaro Herrera takes part in message #3 but says he will not study either patch before 2027. The owner must know REPACK (CONCURRENTLY) and the lock manager.
- Priority: fix design first; audience a sizeable minority; review cost medium. The patch is small (three files, 6 KB) and helps users who repack busy tables with long transactions. The deadlock fallback and the unbounded polling need agreement, and a comparison with message #2, before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
