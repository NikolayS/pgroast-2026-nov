# Design pre-review: REPACK (CONCURRENTLY): do not block the table while waiting for the final lock

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7369](https://commitfest.postgresql.org/patch/7369/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [REPACK (CONCURRENTLY): do not block the table while waiting for the final lock](https://www.postgresql.org/message-id/flat/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com), started 2026-09-30
- Messages read: 4 of 4
- Patch prose read: v1-0001-Do-not-let-REPACK-CONCURRENTLY-block-the-table-wh.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is real and measured, and no surviving objection is fatal. The design leaves polling unbounded when lock_timeout is unset (standards-compat-1), and the decoding slot retains WAL and catalog_xmin for that whole time (replication-ondisk-1). The deadlock fallback covers only waiters on the table lock (maintenance-fit-3). Message #2 offers a competing design, and the thread has not compared the two.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether the fallback through LockHasWaitersRelation() covers every lock REPACK holds while polling, and how the loop interacts with the decoding worker, cannot be judged from the prose.

## The proposal, restated

**Problem.** REPACK (CONCURRENTLY) queues for AccessExclusiveLock at the end, so while a long transaction blocks it, every new query on the table waits behind REPACK. Setting lock_timeout avoids that, but REPACK then fails after all the copying is done. This hits users of busy tables that have long transactions.

**Proposed design.** REPACK makes conditional lock attempts. Between them it applies the changes decoded meanwhile, and it retries every 50 ms, like lazy_truncate_heap(). lock_timeout bounds the polling, and when LockHasWaitersRelation() reports a waiter behind a lock REPACK holds, REPACK falls back to a normal wait so deadlocks are still detected.

**User-visible surface.**
- New wait event REPACK_LOCK_WAIT for the polling phase.
- Changed behavior: REPACK (CONCURRENTLY) no longer queues for ACCESS EXCLUSIVE, so other sessions do not wait behind it.
- lock_timeout now limits how long REPACK keeps polling for the lock.
- Documentation of REPACK rewritten to describe the polling.

**What the author claims.**
- A single-row SELECT waited 14.7 s behind a queued REPACK on master and 0.9 s with the patch.
- With 16 pgbench clients at about 135k tps, REPACK took 1 to 10 s to get the lock, against 0.5 s when queueing.
- The author considers that delay acceptable after hours of copying.
- A lock manager variant that leaves the queue after each deadlock_timeout gets the lock in 0.5 s, but adds 130 lines to lock.c and proc.c.
- The meaning of lock_timeout does not change.

**Thread state.** Antonin Houska questioned using deadlock_timeout and posted his own patch that limits wait time without giving up, because a failed REPACK loses all its work. Alvaro Herrera refuses changes for 19 and will not study either idea before 2027; the author agreed to target 20.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Default REPACK may never finish when lock_timeout is unset

fixable · level 2, rubric item 6 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA runs REPACK (CONCURRENTLY) without lock_timeout on a table with steady short-lock traffic. Conditional attempts keep failing, and REPACK keeps decoding and holding its slot for hours. The DBA notices only when it has not finished and WAL has piled up.

Today REPACK without lock_timeout queues and gets the lock once the blocker ends, so it always finishes. The patch changes that contract with no option to opt out. Message #1 measured 1 to 10 s at 135k tps but gives no upper bound, and a busier table may never see a gap. The LockHasWaitersRelation() fallback only covers waiters behind REPACK's own lock.

Thread: [#1 (shihao zhong, 2026-09-30)](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com)

**Question:** What bounds polling when lock_timeout is unset, and why is there no REPACK option to keep the current queueing behavior?

Counter-check: held. Message #1 says a conditional request succeeds only when nobody holds a lock, and it names no bound other than lock_timeout. The measured 1 to 10 s at 135k tps does not rule out starvation on a busier table.

### 2. Unbounded polling keeps the REPACK decoding slot and its WAL alive

fixable · level 2, rubric item 14 · WAL / replication / standby, catalog changes · `replication-ondisk-1`

A DBA runs REPACK (CONCURRENTLY) on a hot table with lock_timeout unset. Conditional attempts keep failing because some lock is always held, so the logical decoding slot holds back WAL and catalog_xmin while REPACK polls. pg_wal grows until the disk fills, or max_slot_wal_keep_size invalidates the slot and REPACK fails after all the copying.

Message #1 reports 1 to 10 s to get the lock at 135k tps, but gives no upper bound when lock_timeout is unset. Queueing waits only for the holders ahead of it, while polling can lose every attempt to new arrivals. The proposal does not say how long REPACK may keep its slot during polling, or what happens to WAL retention and catalog bloat in that time.

Thread: [#1 (shihao zhong, 2026-09-30)](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com)

**Question:** What bounds polling when lock_timeout is unset, and how is WAL and catalog_xmin retention by the REPACK slot limited while it polls?

Counter-check: held. REPACK (CONCURRENTLY) decodes changes through a slot, and the proposal gives no bound on polling when lock_timeout is unset. Queueing waits only for current holders, so the longer retention of WAL and catalog_xmin is a new cost that the thread does not address.

### 3. Deadlock fallback depends on a check later changes can break

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A later patch makes REPACK hold a lock on another object while it polls. A session waits on that lock while it holds a lock on the table. LockHasWaitersRelation() on the table reports nothing, so REPACK keeps polling and the deadlock detector never runs, and the hang lasts until lock_timeout or manual cancellation.

Message #1 limits the fallback to waiters behind "the lock REPACK holds" on the relation. Any cycle through another lock REPACK holds is outside that check. Correctness then depends on every future change to REPACK's locking also updating the waiter check. A reviewer is likely to miss that, and nothing fails loudly when it is missed.

Thread: [#1 (shihao zhong, 2026-09-30)](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com)

**Question:** Which locks does REPACK hold while polling, and what ensures that a waiter on any of them, now or after later changes, triggers the normal wait path?

Counter-check: held. Message #1 describes the fallback only for waiters behind the lock REPACK holds on the table. The proposal does not list the other locks REPACK holds while polling, so an undetected cycle through one of them cannot be excluded.

### 4. Lock monitoring no longer shows what blocks REPACK

fixable · level 2, rubric item 12 · design, psql UX conventions · `standards-compat-2`

A DBA sees REPACK stuck and runs pg_blocking_pids() on its pid, or looks for an ungranted row in pg_locks, to find the long transaction. With polling there is no waiting lock request, so both return nothing. The DBA cannot identify the session to cancel.

Lock waits in Postgres show up in pg_locks and pg_blocking_pids(), and existing monitoring scripts depend on that. The patch replaces the lock wait with a new wait event, REPACK_LOCK_WAIT, which names no blocker. The documentation hunk does not tell users how to find the blocking session.

Thread: [#1 (shihao zhong, 2026-09-30)](https://www.postgresql.org/message-id/CAGRkXqQTRvmJJBjLmLa0boO1S78nz46s%3DB3htYMFTkSW-FS18Q%40mail.gmail.com)

**Question:** How does a user find the session that blocks a polling REPACK, given that pg_locks and pg_blocking_pids() no longer show the wait?

Counter-check: held. A conditional attempt leaves no ungranted row in pg_locks, so pg_blocking_pids() returns nothing for REPACK. The granted locks on the table are still visible in pg_locks, so the user can find the holder, but the documentation hunk does not say so.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | REPACK keeps its logical decoding slot, and so WAL and catalog_xmin, for as long as it polls. |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-1: Unbounded polling keeps the REPACK decoding slot and its WAL alive |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | see objection standards-compat-2: Lock monitoring no longer shows what blocks REPACK |
| security / privileges | no |  |
| backpatchability | **yes** | The patch changes REPACK behavior; message #3 refuses that for 19, so it targets 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The polling loop in repack.c has its own deadlock fallback, and later locking changes in REPACK must keep it correct. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What bounds polling when lock_timeout is unset, and can a steady stream of short AccessShareLock holders starve REPACK forever?
2. How much WAL and catalog_xmin does the REPACK slot retain while polling on a write-heavy table, and what limits it?
3. Which locks does REPACK hold while polling, and what makes a waiter on any of them trigger the normal wait path?
4. How does this patch compare with the patch in message #2, which limits the wait without letting REPACK give up?
5. How does a user find the session that blocks a polling REPACK, now that pg_blocking_pids() returns nothing for it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Alvaro Herrera takes part in message #3 but says he will not study either patch before 2027. The owner must know REPACK (CONCURRENTLY) and the lock manager.
- **Liability:** The owner signs for REPACK finishing on busy tables, for deadlock detection in the polling loop, and for slot retention while polling, for as long as REPACK (CONCURRENTLY) exists.
- **Author follow-up:** The brief says the author agreed to target 20, which suggests the author stays with the patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is small (three files, 6 KB) and helps users who repack busy tables with long transactions. The deadlock fallback and the unbounded polling need agreement, and a comparison with message #2, before code review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Unbounded polling keeps the decoding slot and WAL pinned. Refuted (not specific): Duplicate of replication-ondisk-1 for slot and WAL retention, and of standards-compat-1 for starvation by weaker lockers.
- `maintenance-fit-2` Lock-queue fix built into one command, separate from Houska's design. Refuted (no failure scenario): The scenario depends on later commands copying the loop, and the proposal does not propose that. The loop follows lazy_truncate_heap(), and applying decoded changes between attempts is specific to REPACK; the comparison with message #2 is pending in the thread.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the thread names no earlier discussion of this idea. Message #2 posts a competing patch that bounds the wait without giving up, and message #3 defers both patches past 19 without judging either one.
- standards-compat: 2 objection(s). Two objections raised; the lock_timeout reuse itself fits existing practice in lazy_truncate_heap().
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL record, catalog or wire protocol. The only cross-boundary state is the logical decoding slot that REPACK already uses, which the objection above covers.
- security: 0 objection(s). No security objection. The patch changes only how REPACK (CONCURRENTLY) gets its final lock, and does not change who may run REPACK or what anyone can see. A user with only SELECT can delay REPACK with a stream of short queries, but under queueing that same user can already block it with one long transaction. Unbounded polling and the WAL kept while it runs belong to the design angle.
- maintenance-fit: 3 objection(s). Committability: Alvaro Herrera knows the area and is engaged, but he defers all study until 2027. The author agreed to target 20.
- already-exists: 0 objection(s). No existing feature lets a user make REPACK (CONCURRENTLY) take its final lock without queueing. lock_timeout loses all the copied work, as message #1 shows, and the proposal reuses the polling pattern of lazy_truncate_heap() instead of adding a new mechanism. Antonin Houska's patch in message #2 is an alternative design under discussion, not an existing feature.
- demand: 0 objection(s). The material shows real demand. The problem is a known property of the lock queue, and the author shows it with numbers (a SELECT waited 14.7 s against 0.9 s). Antonin Houska posted his own patch for the same problem in message #2, so a second person wants it fixed. The existing workaround, lock_timeout, throws away all the copying work, and the cost of the change falls mainly on REPACK itself. The open questions are about design and the choice between the two patches, which other angles cover.

### Precedents retrieved

- P1: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P2: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7039/) (cf, Returned with feedback, 2026-09-13) - judged not relevant
- P3: [REPACK (CONCURRENTLY) doesn't handle invalid indexes](https://commitfest.postgresql.org/patch/7040/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P4: [Add pg_wait_for_lockers() function](https://commitfest.postgresql.org/patch/4480/) (cf, Withdrawn, 2024-07-22) - judged not relevant; reason summarised from its thread
- P5: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P6: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - judged not relevant
- P7: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P8: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P9: [notice lock waits](https://commitfest.postgresql.org/patch/702/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P10: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant; reason summarised from its thread
- P11: [Revert "isolationtester: don't repeat the is-it-waiting query when retrying a step."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dca369320f6023b55feb49f281d394181fc57903) (revert, Reverted, 2016-02-12) - judged not relevant
- P12: [pg_get_domain_ddl: DDL reconstruction function for CREATE DOMAIN statement](https://commitfest.postgresql.org/patch/6983/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P13: [pg_stat_lock: add blocker mode dimension](https://commitfest.postgresql.org/patch/6820/) (cf, Withdrawn, 2026-08-13) - judged not relevant; reason summarised from its thread
- P14: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P15: [fix and document CLUSTER privileges](https://commitfest.postgresql.org/patch/4054/) (cf, Returned with feedback, 2023-01-14) - judged not relevant; reason summarised from its thread
- P16: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P17: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P18: [Remove some unnecessary if-condition](https://commitfest.postgresql.org/patch/2760/) (cf, Withdrawn, 2020-10-22) - judged not relevant
- P19: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - judged not relevant
- P20: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P21: [low wal_retrieve_retry_interval causes missed signals on Windows](https://commitfest.postgresql.org/patch/4118/) (cf, Withdrawn, 2023-02-25) - judged not relevant
- P22: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant
- P23: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant
- P24: [Make relfile tombstone files conditional on WAL level](https://commitfest.postgresql.org/patch/3030/) (cf, Withdrawn, 2021-10-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 145 s; cost reported by the CLI: $1.26
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 77 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:46:32.372Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
