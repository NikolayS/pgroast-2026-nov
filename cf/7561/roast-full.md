# Design pre-review: Call CalculateShmemSize() only once

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7561](https://commitfest.postgresql.org/patch/7561/) (Needs review); authors: Ashutosh Bapat (ashutoshbapat)
- Thread: [Call CalculateShmemSize() only once](https://www.postgresql.org/message-id/flat/CAExHW5ttQauXiYCM4OKL8xKHZrndTN2jMN4WQjm1PkPNuqX_bg%40mail.gmail.com), started 2026-10-30
- Messages read: 7 of 7
- Patch prose read: v20261107-0001-Call-CalculateShmemSize-only-once-when-cre.patch, v20261030-0001-Call-CalculateShmemSize-only-once-when-cre.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived against the change on master, where it is a small refactoring that the listed committer supports. The one surviving objection, maintenance-fit-1, concerns the proposed PG 19 backpatch: the patch changes an exported signature in ipc.h and adds a value in the middle of an enum, and the thread names no future backpatch that needs it. Targeting master only resolves it, and that choice belongs to the committer.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the state transitions on crash restart, the EXEC_BACKEND path and the postgres -C path could not be checked. Only 3 of the 7 messages were visible, so later review points may already answer some questions.

## The proposal, restated

**Problem.** During normal startup the server calls CalculateShmemSize() twice, once in InitializeShmemGUCs() and once in CreateSharedMemoryAndSemaphores(). The author wants to drop the duplicate calculation and make sure GUC reporting and segment creation use the same size. The audience is server developers who maintain this code.

**Proposed design.** Startup code calls CalculateShmemSize() once, after ShmemCallRequestCallbacks(), and passes the result to InitializeShmemGUCs() and CreateSharedMemoryAndSemaphores(). A new shmem_request_state value, SRS_SIZED, sits between SRS_REQUESTING and SRS_INITIALIZING, so the server cannot register new requests after sizing. An Assert catches a second sizing call in the same cycle, and ResetShmemAllocator() resets the state on crash restart.

**User-visible surface.**
- C API: CreateSharedMemoryAndSemaphores() now takes a Size argument.
- C API: InitializeShmemGUCs() receives the calculated size from its caller.
- C API: new SRS_SIZED value in shmem_request_state; extensions that register shared memory requests after sizing are rejected.

**What the author claims.**
- Removes a duplicate shared memory size calculation during startup.
- GUC reporting and segment creation use the same calculated size.
- No request can be registered after sizing, so initialization allocates exactly what was sized.
- An Assert is enough for repeated sizing calls because this is an internal lifecycle invariant.
- Worth backpatching to PG 19 to reduce conflicts in later backpatches, though it is not a bug fix.

**Thread state.** Nathan Bossart (listed committer), Srinath Reddy Sadipiralla and Xuneng Zhou support the patch, and the author squashed both parts into one patch after review. Xuneng Zhou would prefer an ERROR to an Assert, and nobody has decided on the PG 19 backpatch.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatching to PG 19 changes exported API and startup behaviour in a stable branch

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `maintenance-fit-1`

A packager ships a PG 19 minor release with this patch. An extension built against an earlier 19.x minor that calls CreateSharedMemoryAndSemaphores() now passes the wrong arguments. A library that registers a shared memory request between sizing and initialization now fails at server start. The user finds out only after the minor update.

The patch changes the signature of an exported function and adds an enum value in the middle of shmem_request_state. It also makes a previously accepted request timing fail at runtime. The only reason given for PG 19 is fewer conflicts in later backpatches (message #1, message #3), and the author says it is not a bug fix. A committer would likely ask for master only, since stable branches do not usually take refactorings that change exported symbols.

Thread: [#1 (Ashutosh Bapat, 2026-10-30)](https://www.postgresql.org/message-id/CAExHW5ttQauXiYCM4OKL8xKHZrndTN2jMN4WQjm1PkPNuqX_bg%40mail.gmail.com), [#3 (Srinath Reddy Sadipiralla, 2026-10-31)](https://www.postgresql.org/message-id/CAFC%2Bb6pDgRvied%3D1ERdX3GcOAZGV%2B%2BcThZkzeUdgrktzmvc2Pg%40mail.gmail.com), [#4 (Ashutosh Bapat, 2026-11-02)](https://www.postgresql.org/message-id/CAExHW5uc4LodxiiY6S5uOaUNJk9wK-fXt93Ak-g4LJvLEzJ5jg%40mail.gmail.com)

**Question:** Which concrete future backpatch would conflict without this change, and can the PG 19 version keep the old function signature and enum order?

Counter-check: held. The patch changes an exported signature in ipc.h, and the thread gives no concrete future backpatch that justifies this in PG 19. The late-request failure is unshown, but the ABI point stands alone, and targeting master only resolves it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | CreateSharedMemoryAndSemaphores() gains a Size argument in ipc.h, and SRS_SIZED rejects shared memory requests registered after sizing. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 proposes PG 19 for a refactoring that changes an exported signature and tightens the request rules. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which concrete future backpatch would conflict without this change, and can a PG 19 version keep the old function signature and enum order?
2. Does any core path or known extension register a shared memory request between sizing and initialization today, so that SRS_SIZED would now reject it?
3. How do the EXEC_BACKEND child attachment and late allocation paths avoid entering SRS_SIZED?
4. Does postgres -C still report runtime-computed GUCs and exit before shared memory creation under the new call order?
5. Was restart_after_crash tested in a non-assert build, where the Assert against a second sizing call does not run?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Nathan Bossart is listed as committer on the CommitFest entry, reviewed the patch in message #2, and says there that the double call came from his own earlier work.
- **Liability:** The owner signs for startup and crash-restart sizing of shared memory on all platforms, including EXEC_BACKEND, and for any extension that the new SRS_SIZED check rejects. If the patch goes to PG 19, the owner also signs for the ABI change in a stable branch.
- **Author follow-up:** The author posted a revised patch in message #7 that squashes both parts after review, and offered in message #1 to update the patch further.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 9 KB across six startup and shared memory files, benefits developers who maintain this code, and has a committer engaged. The PG 19 backpatch is the only open point, and it can be dropped without changing the master patch.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Backpatching to PG 19 changes C signatures and startup rules in a minor release. Refuted (not specific): Duplicate of maintenance-fit-1. Message #1, not the commit message, says the change is not a bug fix, and no extension is shown to register requests after sizing.
- `replication-ondisk-1` Backpatch to PG 19 changes exported function signature and request rules. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about the signature change and the stricter request rule in PG 19 and also covers the enum order.
- `demand-1` Backpatch to PG 19 moves extension risk into a stable branch. Refuted (not specific): Duplicate of maintenance-fit-1. The who-pays framing adds no new fact beyond the PG 19 signature change and the stricter request rule.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at this idea. In the thread, message #2 has the committer say the double call is his own earlier choice, and nobody points to a rejected discussion of this refactoring.
- standards-compat: 1 objection(s). On master the change is internal and does not touch SQL, GUC semantics, pg_dump or psql. The thread keeps the postgres -C path separate (message #1), so I see no user-facing compatibility objection for master.
- replication-ondisk: 1 objection(s). The patch touches only shared memory sizing at startup. It changes no on-disk format, WAL record, catalog or wire protocol, so the only concern from this angle is the proposed PG 19 backpatch.
- security: 0 objection(s). The patch changes when the postmaster sizes shared memory and adds an internal state. It adds no new privilege and no new user-facing interface. It also gives no non-superuser a new way to see data or use resources. Shared memory requests come only from core and from preloaded libraries that a superuser configured, so the new SRS_SIZED check gives no unprivileged user a path to escalation or denial of service.
- maintenance-fit: 1 objection(s). For master, I found no maintenance or fit objection. The change is a small refactoring of existing startup code, it tightens an existing state machine, adds no per-query cost, and the committer who wrote the original code has engaged and taken the entry.
- already-exists: 0 objection(s). This is an internal refactoring of startup sizing. No existing feature, extension or hook already removes the duplicate CalculateShmemSize() call. SRS_SIZED extends the existing shmem_request_state machine and adds no second mechanism, so this angle has nothing to raise.
- demand: 1 objection(s). For master the demand question does not apply. This is an internal cleanup that Nathan Bossart supports as the listed committer, and it has no user-facing cost. Only the PG 19 backpatch lacks a stated need.

### Precedents retrieved

- P1: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant; reason summarised from its thread
- P4: [Request for votes: Should we remove -Wdeclaration-after-statement?](https://commitfest.postgresql.org/patch/4799/) (cf, Rejected, 2024-03-14) - judged not relevant; reason summarised from its thread
- P5: [Be strict when request to flush past end of WAL in WaitXLogInsertionsToFinish](https://commitfest.postgresql.org/patch/7294/) (cf, Withdrawn, 2026-09-12) - judged not relevant; reason summarised from its thread
- P6: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P7: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P8: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P9: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P10: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P11: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P12: [Simplify standby state machine a bit in WaitForWALToBecomeAvailable()](https://commitfest.postgresql.org/patch/3965/) (cf, Withdrawn, 2023-03-03) - judged not relevant; reason summarised from its thread
- P13: [pg_upgrade allows itself to be run twice (initdb)](https://commitfest.postgresql.org/patch/3872/) (cf, Returned with feedback, 2022-12-01) - judged not relevant
- P14: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [always use runtime checks for CRC-32C instructions](https://commitfest.postgresql.org/patch/4634/) (cf, Withdrawn, 2023-10-31) - judged not relevant
- P16: [Revert hooks for session start and end, take two](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9555cc8d2b02c4191d67ba39f589b39b01735518) (revert, Reverted, 2019-10-02) - judged not relevant
- P17: [Fix runtime errors from -fsanitize=undefined](https://commitfest.postgresql.org/patch/2144/) (cf, Rejected, 2019-08-13) - judged not relevant
- P18: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant
- P20: [some namespace.c refactoring](https://commitfest.postgresql.org/patch/4212/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P21: [Variable Block Size](https://commitfest.postgresql.org/patch/4429/) (cf, Withdrawn, 2024-01-19) - judged not relevant
- P22: [Minimizing calls to SubTransSetParent()](https://commitfest.postgresql.org/patch/3806/) (cf, Withdrawn, 2023-03-22) - judged not relevant
- P23: [psql - sort by size](https://commitfest.postgresql.org/patch/2188/) (cf, Withdrawn, 2019-09-13) - judged not relevant
- P24: [refactor ExecInitPartitionInfo](https://commitfest.postgresql.org/patch/6280/) (cf, Rejected, 2026-07-06) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 96 s; cost reported by the CLI: $1.17
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 66 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:10:14.010Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
