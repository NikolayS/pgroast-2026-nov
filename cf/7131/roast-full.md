# Design pre-review: Fix races in Windows pthread emulation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7131](https://commitfest.postgresql.org/patch/7131/) (Needs review); authors: Harrison Booth (hbooth)
- Thread: [\[PATCH v1\] Fix races in Windows pthread emulation](https://www.postgresql.org/message-id/flat/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), started 2026-07-23
- Messages read: 10 of 10
- Patch prose read: v1-0001-Fix-races-in-Windows-pthread-emulation.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the compare-and-exchange fix. Several people reproduced the hang, and the cost falls only on Windows builds of libpq and ECPG. The one surviving objection, maintenance-fit-1, is a fixable design point: v2 adds a per-compiler atomic load to two copies of the mutex code, and Heikki Linnakangas asked for one primitive in message #8. A committer who knows the Windows port must decide that point and the backpatch range.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and the v2 commit message were not read, so the memory ordering of the new loads on ARM64 and the layout claim for ecpg-pthread-win32.h are unchecked. Messages #4 to #10 are known only from the brief and the reviewer notes.

## The proposal, restated

**Problem.** The Windows pthread mutex emulation in ECPG and libpq lets a waiting thread overwrite the initialized state 1 with state 2. pthread_mutex_unlock then returns EINVAL and leaves the critical section locked, so threaded client code hangs. The ECPG thread/alloc test hangs on native ARM64 and on the AMD64 buildfarm animal hamerkop.

**Proposed design.** Only the thread that changes initstate from 0 with a compare-and-exchange initializes the critical section and publishes state 1. The lock fast path uses an acquire load, and unlock uses a relaxed load after a successful lock. v2 drops the v1 change of pthread_once_t from bool to LONG and leaves type layouts unchanged.

**What the author claims.**
- The existing ECPG thread/alloc test exercises the failing path and reproduced the hang, so no new test is added.
- A deterministic reproducer returns EINVAL on the old code and passes with v2.
- Stress tests passed 800,000 lock/unlock pairs per configuration on native ARM64 MSVC, emulated x64 MSVC and emulated x64 MinGW.
- v2 applies cleanly to REL_17_STABLE and REL_18_STABLE, but neither branch was built.
- The fast-path cost is not benchmarked, and the change touches no non-Windows code or user-facing interface.

**Thread state.** Heikki Linnakangas accepted the v2 approach but asked whether one atomic load works for both MSVC and MinGW instead of ReadAcquire and __atomic_load_n. Alexander Lakhin reported that v2 removes the hang in 600 stress iterations, and backpatching plus the separate pthread_once fix stay open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Ad hoc compiler-specific atomic loads added to two duplicated copies

fixable · level 2, rubric item 12 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later fix or a new Windows toolchain changes the ordering code in the libpq copy but not in the ECPG copy, or only in the MSVC branch. The MinGW branch has less coverage on the buildfarm. The drift shows up months later as a rare hang in client threads.

v2 adds an `#ifdef _MSC_VER` split, with ReadAcquire on MSVC and __atomic_load_n on other compilers, to two copies of the same mutex code. Heikki asked for one primitive for both compilers in message #8, and message #9 leaves that open. Postgres already has its own atomics and barrier abstraction, and the proposal does not say why that is unusable here. CF entry 6314 would merge the libpq copy into src/port (message #1), so the order of the two patches decides how many copies need this code.

Thread: [#1 (Harrison Booth, 2026-07-23)](https://www.postgresql.org/message-id/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), [#8 (Heikki Linnakangas, 2026-10-13)](https://www.postgresql.org/message-id/0f62028a-54bf-427b-9bed-65ee89b34b7c%40iki.fi), [#9 (Harrison Booth, 2026-10-13)](https://www.postgresql.org/message-id/CAAH-eSVLTosOtXkPOnnkwHNRanZ%2BGTioYBoN01LUQv3PyNeKSQ%40mail.gmail.com)

**Question:** Why can the fast path not use the existing Postgres atomics or barrier primitives, or one shared helper, instead of a per-compiler branch in each copy?

Counter-check: held. Heikki asked for one primitive in message #8, and message #9 leaves the per-compiler split open without a benchmark. The two copies predate the patch and port/atomics.h rejects frontend code, so neither point should count as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch touches the ECPG header ecpg-pthread-win32.h; v2 says it keeps type layouts unchanged, after v1 changed pthread_once_t. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug is in released client libraries; v2 applies to REL_17_STABLE and REL_18_STABLE but neither branch was built. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2 adds an MSVC and non-MSVC atomic load branch to two copies of the mutex code, in ECPG and libpq. |
| performance on the hot path | **yes** | The lock and unlock fast paths of the Windows client mutex gain an atomic load, and the cost is not benchmarked. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why can the lock fast path not use one atomic load primitive, or one shared helper, for both MSVC and MinGW in both copies?
2. Which back branches should get this fix, and do REL_17_STABLE and REL_18_STABLE build and pass the ECPG and libpq tests with v2?
3. Should this patch or CommitFest entry 6314 go in first, given that 6314 moves the libpq copy into src/port?
4. Is the pthread_once ordering problem on ARM64 still real after v2, and will a separate patch for it follow?
5. Is CommitFest entry 7255 a duplicate of 7131, and which one should be closed?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas reviewed v2 in the thread and accepted the approach. The owner needs to know the Windows port and the thread-safety code of libpq and ECPG.
- **Liability:** The owner signs for the correctness of a lock-free init state machine in two client libraries on MSVC and MinGW, on x64 and ARM64, in every branch that gets the fix. Races of this kind show up rarely and late, and MinGW has less buildfarm coverage.
- **Author follow-up:** The author posted v2 after review and dropped the pthread_once change that was questioned. No follow-up patch for pthread_once is posted.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch fixes a reproduced hang for threaded Windows clients of libpq and ECPG, and it also removes a test hang seen in CI and on the buildfarm animal hamerkop. It is about 4 KiB in three client-side files, adds no user-visible surface, and has one open point about the atomic load primitive.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at the same or a similar idea, and the thread mentions no earlier discussion of this race. The thread refers only to CommitFest entry 6314, a pending refactor that moves the libpq code into src/port without changing this state machine (message #1), and to entry 7255, created by a reviewer to track this same patch (message #3). Neither is a prior rejection, so this angle has nothing to object to.
- standards-compat: 0 objection(s). Nothing in the user-facing contract changes: no SQL, GUC, psql, pg_dump or driver-visible behaviour, and v2 keeps the existing type layouts, including in ecpg-pthread-win32.h. Backpatch target branches are still open, and REL_17_STABLE and REL_18_STABLE were not built (message #7), but that is a testing step and does not change the design.
- replication-ondisk: 0 objection(s). v2 has no persistent or cross-node state. It changes only the in-process Windows mutex emulation in libpq and ECPG, with no on-disk, WAL, catalog or protocol change. Message #7 says v2 keeps the existing type layouts, which drops the v1 pthread_once_t change that message #4 questioned for backpatching. Without the code, I could not check whether the installed ecpg-pthread-win32.h exposes anything that the new atomic loads would affect for applications built against older headers.
- security: 0 objection(s). The patch fixes a client-side race in the Windows pthread mutex emulation in ECPG and libpq. It adds no privilege, role, setting, data path or interface that an unprivileged user could use, and it changes no server-side behaviour. A hang in a threaded client is a bug in the client's own process, so it does not give a denial-of-service path against other users.
- maintenance-fit: 1 objection(s). The fix targets a real hang that several people reproduced, and its cost falls only on Windows client builds. The objection above is about how the code is placed, not whether to fix the bug.
- already-exists: 0 objection(s). This is a fix for a real hang that two independent reporters and the buildfarm reproduced, and no existing feature or extension removes the need for it. A committer might suggest native Windows primitives such as SRWLOCK or InitOnceExecuteOnce instead of the custom init state, but that is a design choice for a possibly backpatched bugfix and does not argue against the fix.
- demand: 0 objection(s). This is a bug fix for a hang in shipped client libraries, so demand is shown. Independent reports: Nazir Bilal Yavuz hit it twice in CI (messages #2, #3), the buildfarm animal hamerkop failed twice this year (message #5), and Alexander Lakhin reproduced it on x86_64 (messages #5, #10). The cost falls only on Windows builds of ECPG and libpq, and no user-facing surface changes.

### Precedents retrieved

- P1: [Use internal pthreads reimplementation only when building with MSVC](https://commitfest.postgresql.org/patch/2526/) (cf, Returned with feedback, 2020-07-31) - judged not relevant; reason summarised from its thread
- P2: [Make ECPG locale usage thread-safe](https://commitfest.postgresql.org/patch/5166/) (cf, Returned with feedback, 2026-02-09) - judged not relevant; reason summarised from its thread
- P3: [Avoid use deprecated Windows Memory API](https://commitfest.postgresql.org/patch/3893/) (cf, Rejected, 2023-03-20) - judged not relevant; reason summarised from its thread
- P4: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P5: [Revert "Tidy up locale thread safety in ECPG library."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3c8e463b0d885e0d976f6a13a1fb78187b25c86f) (revert, Reverted, 2025-03-28) - judged not relevant
- P6: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P7: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P8: [meson: Adjust test timeout for Valgrind builds](https://commitfest.postgresql.org/patch/6646/) (cf, Withdrawn, 2026-06-10) - judged not relevant; reason summarised from its thread
- P9: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant; reason summarised from its thread
- P10: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P11: [Fix error message for statement_timeout and fix flaky test](https://commitfest.postgresql.org/patch/5072/) (cf, Withdrawn, 2024-06-25) - judged not relevant; reason summarised from its thread
- P12: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant; reason summarised from its thread
- P13: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3560/) (cf, Returned with feedback, 2022-02-26) - judged not relevant
- P14: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3561/) (cf, Returned with feedback, 2022-12-01) - judged not relevant
- P15: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant
- P16: [Clarify need for memory barriers in bgworkers](https://commitfest.postgresql.org/patch/69/) (cf, Rejected, 2015-02-03) - judged not relevant
- P17: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P18: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/4309/) (cf, Returned with feedback, 2023-09-19) - judged not relevant
- P19: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P20: [Fix MINGW build on ARM64 on Windows](https://commitfest.postgresql.org/patch/5610/) (cf, Returned with feedback, 2025-04-07) - judged not relevant
- P21: [pgbench - - break out timing data for initialization phases](https://commitfest.postgresql.org/patch/1308/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P22: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P24: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 85 s; cost reported by the CLI: $1.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 48 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:22:39.751Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
