# Design pre-review: Fix a file descriptor leak when using io_method=io_uring

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6617](https://commitfest.postgresql.org/patch/6617/) (Needs review); authors: Lucas DRAESCHER (draescherl)
- Thread: [\[Bug Report + Patch\] File descriptor leak when io_method=io_uring](https://www.postgresql.org/message-id/flat/zHN8Rb7syWYDBPUhCICxKLr_tQmFLg0T024irxVVb6RSSicZcqcA0eT73z2PQy7JmzzqnUzX0Hi3P2dPKRvmNx_jbAqSCLyqJvWjXEypyAI%3D%40draescher.fr), started 2026-03-18
- Messages read: 22 of 22
- Patch prose read: v4-0001-Release-io_uring-resources-on-shmem-exit.patch, v1-0001-Release-io_uring-resources-on-shmem-exit.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproduction, several reviewers confirmed the leak, and the fix follows the existing pgaio_worker_die() pattern in method_worker.c. A committer reports the v1-based fix pushed to master and 18, which leaves IoMethodOps unchanged. Confidence is medium because only 3 of 22 messages and no code were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and messages #4 to #22 were not read, so the callback's behaviour in the postmaster and the pushed commit could not be checked. The follow-up that depends on liburing internals was not judged.

## The proposal, restated

**Problem.** With io_method=io_uring, the postmaster never releases its io_uring instances, so each crash restart after a killed backend doubles the io_uring fd count. Repeated crashes, or one crash with large max_connections, can then stop the restart with "Too many open files" or "Cannot allocate memory", and the server shuts down. Postgres 18 and master are affected.

**Proposed design.** v1 registers an on_shmem_exit() callback in pgaio_uring_shmem_init() that calls io_uring_queue_exit() for each ring, following pgaio_worker_die() in method_worker.c. v2 to v4 instead add a shmem_cleanup callback to IoMethodOps, registered in AioShmemInit(). The committer used a revised v1 for master and REL_18_STABLE, and reports it pushed.

**User-visible surface.**
- C API: new shmem_cleanup member of IoMethodOps in the installed server header aio_internal.h (v2 to v4 only, not in the committed v1-based fix).
- Server log: the restart no longer fails with "could not setup io_uring queue: Too many open files" after crash restarts.

**What the author claims.**
- The leak reproduces on Postgres 18.0 to 18.3 and master, on kernels 6.8 to 6.19 with liburing 2.5 to 2.13.
- Only the postmaster runs the callback, because child processes reset the postmaster's exit callbacks.
- v3 and v4 break the Postgres 18 ABI if backpatched, because the new field sits in the middle of IoMethodOps.
- The author first hit the bug through ulimit -l, which gave "Cannot allocate memory" at restart.

**Thread state.** Several reviewers confirmed the leak and the fix, and Fujii Masao reports he pushed the v1-based patch to master and 18. Still open is a follow-up so cleanup uses postmaster-local copies of ring descriptors, since shared memory may be corrupted; Fujii Masao's draft depends on liburing internals.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v2 to v4 add a shmem_cleanup member to IoMethodOps in aio_internal.h; the committed v1-based fix does not. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix targets REL_18_STABLE; v3 and v4 would break the Postgres 18 ABI, so the v1 shape was used. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The postmaster now calls io_uring_queue_exit() at crash restart, and the planned follow-up depends on liburing internals. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which commit and which branches carry the fix, so the CommitFest entry can be closed as committed?
2. Does liburing allow io_uring_queue_exit() on a postmaster-local copy of struct io_uring, or does the follow-up need another way to keep the ring descriptors?
3. Can a test kill a backend with io_method=io_uring and check that the postmaster's io_uring fd count stays the same after the crash restart?
4. Will the shmem_cleanup member of IoMethodOps still be proposed for master, and which second IO method would use it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao is listed as committer on the CommitFest entry and reports the v1-based fix pushed. The follow-up needs someone who knows the AIO io_uring code and postmaster crash restart.
- **Liability:** The owner signs for the postmaster's exit and crash-restart path with io_method=io_uring on master and 18, including cleanup that reads ring descriptors from shared memory a crashed backend may have corrupted, until the follow-up lands.
- **Author follow-up:** The author posted v2 to v4 after review and set the entry back to Needs review on 2026-09-09, so the author stays engaged.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is a small change in one file for a confirmed leak that can stop a crash restart with io_method=io_uring, and it is reported pushed. What remains is to close the entry and review the follow-up for postmaster-local ring descriptors as a separate patch.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this idea, and the thread does not point to an earlier discussion of it. Several reviewers confirmed the bug fix and a committer reports it pushed (message #18), so a prior-attempts objection has nothing to stand on.
- standards-compat: 0 objection(s). The fix adds no SQL syntax, GUC, catalog object, dump state or psql behaviour, and the only change users see is that crash restarts now succeed. The one compatibility risk, the IoMethodOps ABI break in v3 and v4, was settled in messages #9, #12 and #14 when the committer chose the v1-based fix, which leaves the struct unchanged, for master and 18.
- replication-ondisk: 0 objection(s). The fix changes no on-disk format, WAL record, catalog, replication or wire-protocol state; it only releases postmaster-local io_uring fds at crash restart. The committed v1 shape leaves IoMethodOps unchanged, so the v18 ABI is safe. The risk of trusting ring descriptors in shared memory that a crashing backend may have corrupted was raised in message #16, and a follow-up is in progress (messages #17 and #21).
- security: 0 objection(s). The fix adds no new privileges and exposes no data; message #10 shows the leaked ring fds are FD_CLOEXEC and so never reach COPY FROM PROGRAM or untrusted PL children. The postmaster's use of ring descriptors from shared memory (message #16) needs prior memory corruption, gains no privilege, and is already tracked as a follow-up.
- maintenance-fit: 0 objection(s). This is a small bug fix for a real crash-restart failure, and a committer has already pushed it to master and 18. From the maintenance angle it adds no hot-path cost, no new API in the committed form, and no new external dependency beyond the existing liburing calls. The open follow-up for postmaster-local ring descriptors, which depends on liburing internals, is a separate proposal and should be judged when it is posted.
- already-exists: 0 objection(s). No existing feature does the job. Postgres has no way to release io_uring instances in the postmaster across a crash restart, so the server can fail to restart. The fix reuses the existing on_shmem_exit() pattern from method_worker.c and adds no second mechanism. An extension or hook cannot fix a leak inside core AIO setup.
- demand: 0 objection(s). The leak is a confirmed bug fix, not a feature. Four reviewers reproduced it on several kernels, and message #10 shows one crash with max_connections = 1000 stops the server. The author hit it in production (message #12), so demand is not in question.

### Precedents retrieved

- P1: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P2: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant; reason summarised from its thread
- P4: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P5: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant; reason summarised from its thread
- P6: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P7: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P8: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant; reason summarised from its thread
- P9: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P10: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P11: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P12: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P13: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P14: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P16: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P17: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P18: [Revert "Replace PostmasterRandom() with a stronger way of generating randomness."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faae1c918e8aaae034eaf3ea103fcb6ba9adc5ab) (revert, Reverted, 2016-10-18) - judged not relevant
- P19: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P20: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant
- P21: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P22: [Add MAIN_RELATION_CLEANUP and TOAST_TABLE_CLEANUP options to VACUUM](https://commitfest.postgresql.org/patch/2426/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P23: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P24: [Cygwin cleanup](https://commitfest.postgresql.org/patch/3781/) (cf, Withdrawn, 2023-03-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 77 s; cost reported by the CLI: $1.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:56:17.424Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
