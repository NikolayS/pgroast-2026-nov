# Design pre-review: Fix races in Windows pthread emulation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7255](https://commitfest.postgresql.org/patch/7255/) (Needs review); authors: Harrison Booth (hbooth)
- Thread: [\[PATCH v1\] Fix races in Windows pthread emulation](https://www.postgresql.org/message-id/flat/CAAH-eSXFXSEtXpb4kUAGOKKdm3o3y-yWfjsJ7R7zUL6g-swuzw%40mail.gmail.com), started 2026-07-23
- Messages read: 10 of 10
- Patch prose read: v1-0001-Fix-races-in-Windows-pthread-emulation.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b. The hang is confirmed by Nazir's CI, by hamerkop and by an independent reproducer. The one surviving objection, maintenance-fit-1, is a fixable design point: v2 keeps separate MSVC and MinGW load paths in both the ECPG and libpq copies, and Heikki's request in message #8 for one primitive is not settled. The backpatch decision also needs a committer.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot confirm that the acquire load is sufficient on ARM64 or that no changed logic in ecpg-pthread-win32.h is compiled into applications. The back branches were not built.

## The proposal, restated

**Problem.** The ECPG thread/alloc test can hang until the Meson timeout on Windows. A waiting thread can overwrite the initialized mutex state 1 with 2, so pthread_mutex_unlock returns EINVAL and leaves the critical section locked. Nazir saw it in CI, and the buildfarm animal hamerkop hit it on x86_64.

**Proposed design.** v2 claims initialization with InterlockedCompareExchange only when initstate is 0, and only that thread initializes the critical section and publishes state 1. The lock fast path uses an acquire load (ReadAcquire on MSVC, __atomic_load_n on MinGW), and unlock uses a relaxed load. The pthread_once change from v1 was dropped and will be handled separately.

**What the author claims.**
- The race is not specific to ARM64, though ARM64 is where the author reproduced it.
- A deterministic reproducer returns EINVAL on the old code and passes with v2.
- Stress tests passed 800,000 lock/unlock pairs per configuration on native ARM64 MSVC, emulated x64 MSVC and emulated x64 MinGW.
- v2 applies cleanly to REL_17_STABLE and REL_18_STABLE, but neither branch was built.
- The fast path avoids a read-modify-write, but the author has not benchmarked any variant.

**Thread state.** Heikki accepted the acquire-load approach and asked whether one primitive could serve both MSVC and MinGW. Alexander's reproducer passed 300 iterations with v2, and the pthread_once fix and the backpatch decision are still open.

## 2. Prior attempts

### Fix MINGW build on ARM64 on Windows (related)

- **Link:** [https://commitfest.postgresql.org/patch/5610/](https://commitfest.postgresql.org/patch/5610/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAHXCYb2OjNHtoGVKyXtXmw4B3bUXwJX6M-Lcp1KcMCRUMLOocA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAHXCYb2OjNHtoGVKyXtXmw4B3bUXwJX6M-Lcp1KcMCRUMLOocA@mail.gmail.com)
- **Date:** decided 2025-04-07; started 2025-02-01
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Andrew Dunstan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andrew Dunstan committed the patch, then reverted it after the buildfarm animal fairywren started failing, which Tom Lane reported. Andrew Dunstan confirmed the commit caused the failures. A new submission must show that the change does not break x86_64 builds with any msys2 toolchain.
- **Reason (quote):** "fairywren has been failing rather horribly since this went in [1]." ([message](https://www.postgresql.org/message-id/3312149.1744001936%40sss.pgh.pa.us))
- **What is different this time:** P16 changed setjmp handling for MinGW toolchains and was reverted after an msys2 buildfarm animal broke. This patch fixes a mutex race; the only link is its MinGW-only branch, tested only on emulated x64 MinGW.


## 3. Objections a reviewer is likely to raise

### 1. Compiler-specific atomic loads added to two duplicated mutex copies

fixable · level 2, rubric item 12 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later fix to the Windows mutex state machine changes the MSVC ReadAcquire branch in libpq and misses the MinGW __atomic_load_n branch, or the ECPG copy. Nothing fails at build time. Users of the build that was missed see the hang again, and it shows up only as a rare timeout on CI or the buildfarm.

v2 adds an #ifdef _MSC_VER split in both src/interfaces/ecpg and src/interfaces/libpq, so the same ordering logic now exists in four places. Postgres already has an atomics layer in port/atomics.h, and the proposal does not say why it cannot be used here. Heikki asked for one primitive for both compilers in message #8. Message #9 keeps the split to avoid a compare-exchange on the fast path, but no benchmark exists, and message #4 calls ecpg performance on Windows a minor concern.

Thread: [#4 (Heikki Linnakangas, 2026-10-05)](https://www.postgresql.org/message-id/e39c8c80-5554-4bc1-a232-aaa552e556d9%40iki.fi), [#8 (Heikki Linnakangas, 2026-10-13)](https://www.postgresql.org/message-id/0f62028a-54bf-427b-9bed-65ee89b34b7c%40iki.fi), [#9 (Harrison Booth, 2026-10-13)](https://www.postgresql.org/message-id/CAAH-eSVLTosOtXkPOnnkwHNRanZ%2BGTioYBoN01LUQv3PyNeKSQ%40mail.gmail.com)

**Question:** Can the fast-path load use one existing primitive, such as InterlockedCompareExchange or a port/atomics.h read, in one place shared by ECPG and libpq?

Counter-check: held. Heikki asked for one primitive in message #8 and has not accepted the split, which rests on an unbenchmarked fast-path cost. The port/atomics.h suggestion must be dropped because that header rejects frontend code such as libpq and ECPG, and the two copies predate this patch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch touches ecpg-pthread-win32.h; v2 leaves the mutex and pthread_once_t layouts unchanged. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | v2 applies to REL_17_STABLE and REL_18_STABLE but was not built there, and the backpatch decision is open. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2 adds an MSVC and a MinGW load path to each of the two mutex copies in ECPG and libpq. |
| performance on the hot path | **yes** | The mutex lock fast path in libpq and ECPG on Windows gains an acquire load; no variant was benchmarked. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the lock fast path use one primitive for both MSVC and MinGW, such as InterlockedCompareExchange, and what does that cost compared with the split in v2?
2. Which back branches does the patch target, and has v2 been built and tested on REL_17_STABLE and REL_18_STABLE?
3. When will the separate pthread_once fix be posted, and does native ARM64 stay exposed to the ordering problem from message #1 until then?
4. Will the deterministic reproducer be added as a regression test?
5. Should this patch or CommitFest entry 6314 land first, and does the fix then need to exist in only one copy on master?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki reviewed the approach in messages #4 and #8; the owner needs to know Windows interlocked primitives and the libpq and ECPG thread code.
- **Liability:** The owner signs for lock ordering on MSVC and MinGW, on x64 and ARM64, in two copies, and on every back branch that receives the fix. A mistake shows only as a rare client hang.
- **Author follow-up:** The author posted v2 in response to review, dropped the pthread_once change on request, and said a separate pthread_once fix will follow.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch fixes a confirmed deadlock in threaded libpq and ECPG clients on Windows, seen in CI and on hamerkop. It is about 4 KiB across three files, adds no user-visible surface, and a committer is already reviewing it.

## Appendix

### Refuted in the counter-check

- `already-exists-1` First-use init state machine kept where SRWLOCK_INIT gives static initialization. Refuted (no failure scenario): The scenario is a hypothetical later mistake, already covered by maintenance-fit-1, and v2 itself fails nowhere in it. The first-use init design predates this bug fix, and the objection accepts v2 for back branches, so it asks for a separate master-only rewrite.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent tried to fix this race, and none was rejected for a reason that applies here. P1 proposed switching MinGW to winpthreads, which is a different idea, and the thread does not point to any earlier discussion. The bug is confirmed by CI, by hamerkop and by an independent reproducer, and v2 answers Heikki's cost and backpatch concerns.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. v2 changes no SQL, syntax, GUC, psql behaviour, pg_dump output or wire protocol, and it leaves the pthread_once_t and mutex type layouts unchanged. That removes the one compatibility question raised in the thread, Heikki's backpatch concern in message #4. Whether the change is safe for back branches can only be checked by building them, and that is outside this angle.
- replication-ondisk: 0 objection(s). The patch creates no persistent or cross-node state. It does not touch on-disk format, WAL, replication, catalogs or the wire protocol, and it changes only in-process mutex initialization in ecpglib and libpq on Windows. The one ABI question was Heikki's backpatch concern about changing pthread_once_t in message #4, and v2 leaves pthread_once and all type layouts unchanged (message #7), which settles it. Without the code I could not check one point: whether any changed logic lives in the ecpg-pthread-win32.h header and is compiled into applications, which would matter for a backpatch.
- security: 0 objection(s). The patch fixes a race in the client-side Windows mutex emulation in ECPG and libpq. It adds no privileges, settings, catalog objects or server code paths, and an unprivileged user gains no new reach. The bug is a self-inflicted client hang, so I see no security or privilege aspect to object to.
- maintenance-fit: 1 objection(s). There is no fundamental objection. The patch fixes a real deadlock seen in CI and on hamerkop, touches no non-Windows code, and a committer reviewed it in messages #4 and #8. The only remaining point is how the code is shaped for long-term maintenance.
- already-exists: 1 objection(s). The bug is real, it reproduces on x86_64 and ARM64, and no existing feature or extension fixes it. A narrow back-branch fix is reasonable. The one objection above is about the design for master only.
- demand: 0 objection(s). This is a bug fix, and the thread shows independent reports of the hang: Nazir's CI twice (messages #2, #3), buildfarm animal hamerkop twice this year, and a local x86_64 reproduction (message #5). Every Windows build of libpq and ECPG is affected, so the demand angle has no objection.

### Precedents retrieved

- P1: [Use internal pthreads reimplementation only when building with MSVC](https://commitfest.postgresql.org/patch/2526/) (cf, Returned with feedback, 2020-07-31) - used in the review; reason summarised from its thread
- P2: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P3: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P4: [Revert "Tidy up locale thread safety in ECPG library."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3c8e463b0d885e0d976f6a13a1fb78187b25c86f) (revert, Reverted, 2025-03-28) - judged not relevant
- P5: [Atomic pgrename on Windows](https://commitfest.postgresql.org/patch/1386/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P6: [Avoid pallocs in async.c's "critical section"](https://commitfest.postgresql.org/patch/6248/) (cf, Withdrawn, 2026-01-16) - judged not relevant; reason summarised from its thread
- P7: [Avoid use deprecated Windows Memory API](https://commitfest.postgresql.org/patch/3893/) (cf, Rejected, 2023-03-20) - judged not relevant; reason summarised from its thread
- P8: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3560/) (cf, Returned with feedback, 2022-02-26) - judged not relevant
- P9: [Revert change of ecpglib major version](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74f2a8aa27cb7bd2dbab3ed58e6b5c239a6b4a31) (revert, Reverted, 2019-09-19) - judged not relevant
- P10: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3561/) (cf, Returned with feedback, 2022-12-01) - judged not relevant
- P11: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant; reason summarised from its thread
- P12: [Make ECPG locale usage thread-safe](https://commitfest.postgresql.org/patch/5166/) (cf, Returned with feedback, 2026-02-09) - judged not relevant; reason summarised from its thread
- P13: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/4309/) (cf, Returned with feedback, 2023-09-19) - judged not relevant
- P14: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant; reason summarised from its thread
- P15: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P16: [Fix MINGW build on ARM64 on Windows](https://commitfest.postgresql.org/patch/5610/) (cf, Returned with feedback, 2025-04-07) - used in the review; reason summarised from its thread
- P17: [Fix missed parallel safety checks and wrong parallel safety](https://commitfest.postgresql.org/patch/3094/) (cf, Withdrawn, 2021-06-21) - judged not relevant
- P18: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P19: [GUC to improve SECURITY INVOKER safety](https://commitfest.postgresql.org/patch/4121/) (cf, Withdrawn, 2023-02-02) - judged not relevant
- P20: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P21: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P22: [Close race in DROP TABLESPACE on Windows](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Tablespaces) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P24: [Revert "Avoid race condition between "GRANT role" and "DROP ROLE"".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8abbbbae610cd62faad4f9ec18e7cfa908593796) (revert, Reverted, 2025-09-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 129 s; cost reported by the CLI: $1.20
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 55 text field(s) checked, 3 finding(s), 3 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:33:22.213Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
