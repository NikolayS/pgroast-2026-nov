# Design pre-review: Add configure-time choice of either port/atomics.h or <stdatomic.h> for atomic operations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7005](https://commitfest.postgresql.org/patch/7005/) (Needs review); authors: Thomas Munro (macdice), Greg Burd (gregburd)
- Thread: [Trying out <stdatomic.h>](https://www.postgresql.org/message-id/flat/CA%2BhUKGKFvu3zyvv3aaj5hHs9VtWcjFAmisOwOc7aOZNc5AF3NA%40mail.gmail.com), started 2025-11-10
- Messages read: 35 of 35
- Patch prose read: v2-0001-ci-Use-Visual-Studio-2022-for-stdatomic.h.patch, v2-0002-Enable-Visual-Studio-s-stdatomic.h-support.patch, v2-0003-Redefine-port-atomics.h-on-top-of-stdatomic.h.patch, v2-0004-Use-atomics-API-to-implement-spinlocks.patch, v2-0005-Remove-configure-meson-checks-for-atomics.patch, v1-0001-Add-some-missing-include-limits.h.patch, v1-0002-Redefine-port-atomics.h-on-top-of-stdatomic.h.patch, v1-0003-Use-atomics-API-to-implement-spinlocks.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, but the design is not settled. The proposal offers v11 and v12 together and does not say which one is for commit. Each has an open design point: v11 has no tester or exit rule and changes the slock_t layout by build option (maintenance-fit-1, replication-ondisk-1); v12 removes the old code on untested platforms (maintenance-fit-3, standards-compat-2). Both rely on a seq_cst read that hides a tuple loss of unknown cause (maintenance-fit-2).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only the first 3 of 35 messages and the v1 and v2 patch prose were read; no code was read. Layout sizes under both settings, the barrier implementation and the generated code per architecture could not be checked.

## The proposal, restated

**Problem.** Postgres maintains its own atomics and spinlock code, written per architecture and per compiler. Greg Burd wants to reduce that code and possibly improve performance by using the compiler's C11 atomics. Peter Eisentraut names the end goal: no longer maintaining an in-tree atomics library at all.

**Proposed design.** v11 adds a second atomics implementation built on <stdatomic.h>, chosen at build time and off by default; the old code stays untouched. Spinlocks become a pg_atomic_uint32, and barriers use a signal fence plus a thread fence. v12 is the alternative: one patch that makes <stdatomic.h> the only implementation and removes arch-*.h, generic*.h, fallback.h and s_lock.h.

**User-visible surface.**
- New meson option -Duse_stdatomic=auto|yes|no, default no (v11).
- New configure option --with-stdatomic=auto|yes|no, default no (v11).
- v12 makes configure and meson fail when 32- and 64-bit atomics are not always lock-free.
- pg_atomic_read_u32/u64 get seq_cst ordering, stronger than their documented "no barrier semantics"; extensions use these.
- pg_atomic_flag and slock_t change internal representation; the public pg_atomic_* and SpinLock* APIs stay the same.

**What the author claims.**
- No measurable regression on x86-64 (within about 0.5%), and parity to slightly faster on aarch64 read-only pgbench.
- Builds and passes check-world on x86-64, aarch64, RISC-V, FreeBSD and Windows ARM64 with MSVC.
- With relaxed reads, join_hash loses a tuple on rv64 with gcc 13.3; seq_cst reads fix it, and the cause is unknown.
- Master already requires Visual Studio 2022, so v12 does not raise the MSVC baseline.
- The seq_cst read cost on ARM64, rv64 and POWER, across about 200 call sites, is not benchmarked.

**Thread state.** Tom Lane asked for an alternative rather than a replacement; Peter Eisentraut asked what the long-term plan is and what testing a default-off option is for. Shihao Zhong reviewed v8 to v10 on macOS and found three bugs, now fixed; no committer has replied since Greg posted v11 and v12 together.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-11-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Seq_cst read is a workaround for a bug whose cause is unknown

fixable · level 2, rubric item 9 · design, performance on the hot path, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

Under v12, a POWER or rv64 build pays an hwsync or two fences on each of about 200 pg_atomic_read_u32/u64 call sites, including LWLock and buffer paths. Nobody can safely relax this later, because no one knows which caller needs the ordering. A new compiler then exposes the hidden bug in some other form.

Message #35 says relaxed reads lose a join_hash tuple on rv64 with gcc 13.3 only, and the cause is not found. Message #24 also reports that a pg_read_barrier in the consumer did not help. So the stronger ordering hides a missing ordering somewhere, and the project would keep it forever against the documented "no barrier semantics" contract. The read cost is measured on aarch64 only; message #35 says it is not benchmarked on rv64 or POWER.

Thread: [#24 ("Greg Burd", 2026-09-10)](https://www.postgresql.org/message-id/34101612-a419-4811-a672-f9e55b4fbec2%40app.fastmail.com), [#35 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me)

**Question:** Which caller relies on ordering that pg_atomic_read_u32 does not promise, and what does seq_cst cost on POWER and rv64 under a contended pgbench run?

Counter-check: held. Message #35 itself says the cause of the rv64 tuple loss is unknown and the seq_cst read cost on rv64 and POWER is not benchmarked. Nothing in the thread answers either point.

### 2. Default-off second atomics implementation has no tester and no exit plan

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch adds a new pg_atomic_* operation or changes barrier use and updates only the traditional headers, which every default build uses. The stdatomic path then stops building or behaves differently. Nobody finds out until someone tries -Duse_stdatomic=yes months later.

v11 keeps arch-*.h, generic*.h and s_lock.h and adds stdatomic_impl.h beside them, so each atomics change must be made twice. The proposal names no CI task or buildfarm animal that builds the opt-in path. In message #25 Peter asks what testing the option is for and what happens when it is complete. Message #26 answers only with "cut over mid-cycle if all looks good", which gives no criteria and no date. No committer has replied since v11 was posted, and co-author Thomas Munro has not posted since message #15.

Thread: [#15 (Thomas Munro, 2025-11-23)](https://www.postgresql.org/message-id/CA%2BhUKGJSx124Y1gFJi2y_eRFpO%3DUbcdi71-iEn6nmBQjbZFZ%2Bg%40mail.gmail.com), [#25 (Peter Eisentraut, 2026-09-14)](https://www.postgresql.org/message-id/649b0015-f345-42fe-b1ec-a23beadeec6e%40eisentraut.org), [#26 ("Greg Burd", 2026-09-14)](https://www.postgresql.org/message-id/cae320c9-6882-47c0-8d06-8ff8ab48aafb%40app.fastmail.com), [#35 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me)

**Question:** Which CI tasks or buildfarm animals will build the opt-in path by default, and what results would trigger removing one implementation?

Counter-check: held. Peter asked in message #25 what testing the option is for and what follows. Message #26 names no tester, no criteria and no date, and cfbot builds only the default path.

### 3. V12 hands atomics to compilers on untested platforms, with no fallback

fixable · level 2, rubric item 15 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-3`

After v12 is committed, a POWER, s390x or illumos buildfarm animal shows rare corruption or a hang, or an MSVC update changes /experimental:c11atomics. The project has deleted the hand-written code it would compare against. Back branches keep the old code, so fixes no longer backpatch cleanly.

In message #20 Tom asked for an alternative and not a replacement, because stdatomic codegen is not under project control. v12 removes the fallback anyway, and message #35 says the buildfarm "would be the first real test" on POWER, s390x and Solaris/illumos. MSVC still calls the needed flag experimental. A committer would likely ask for v11-style evidence on those platforms before deleting anything.

Thread: [#8 (Thomas Munro, 2025-11-20)](https://www.postgresql.org/message-id/CA%2BhUKGKfNuXYVKT7WPpKTNYTgPduzu0%3DG5yFEMju_4kbW0ybOQ%40mail.gmail.com), [#20 (Tom Lane, 2026-03-23)](https://www.postgresql.org/message-id/3532551.1774275186%40sss.pgh.pa.us), [#35 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me)

**Question:** What codegen comparison or test results on POWER, s390x and illumos show the stdatomic path matches the existing code before the old headers are removed?

Counter-check: held. Tom asked in message #20 for an alternative and no removal, and message #35 says POWER, s390x and Solaris/illumos are untested. The backpatch remark is generic and should not be used as support.

### 4. Build option changes slock_t layout without an extension ABI check

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A user runs a server built with -Duse_stdatomic=yes or auto and loads an extension binary built against default headers. A shared-memory struct holding slock_t or pg_atomic_flag gets different offsets on each side. The extension then reads and writes the wrong bytes under load, and nothing reports a mismatch at load time.

Message #34 says slock_t becomes a pg_atomic_uint32, and the brief says pg_atomic_flag changes representation. On x86 the traditional slock_t is one byte. With v11 the layout of extension-visible shared-memory structs therefore depends on a configure choice, and with "auto" on the build host's probe result. The proposal does not say whether the option goes into the PG_MODULE_MAGIC ABI fields or whether both paths keep identical sizes. v12 avoids this because it has only one layout per major version.

Thread: [#34 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/7gJmxtbunjsr2_04YEK5pN3XH0ZqYY8E2HQxFZFEsCUxScihXgxQyL5KNQ6nCO_X1bOLgsiCcPX1iy7sB9BSQH8MoqdB4-wgRTYAYXo0UR4%3D%40burd.me), [#35 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me)

**Question:** Do slock_t and pg_atomic_flag keep the same size and alignment under both settings, and if not, how does loading a mismatched extension fail?

Counter-check: held. Message #34 confirms slock_t becomes a pg_atomic_uint32 on the opt-in path, while the traditional x86 slock_t is one byte. The material shows no load-time check for an extension built with the other setting.

### 5. v12 turns build success on some platforms into a configure failure

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `standards-compat-2`

A packager for a 32-bit target whose compiler emulates 64-bit atomics with locks upgrades to the release with v12. configure now stops with an error, where master builds through the existing fallback code. The packager finds out at build time, with no supported way to continue.

Message #35 says configure and meson now fail unless 32- and 64-bit atomics are always lock-free. The v1 commit message planned to let the compiler emulate 64-bit atomics with locks on armv7, so the requirement is narrower than the earlier design. Message #35 also says POWER, s390x, Solaris and illumos are untested and that "the buildfarm would be the first real test". The proposal does not list which supported platforms and compilers meet the new requirement and which lose support.

Thread: [#1 (Thomas Munro, 2025-11-10)](https://www.postgresql.org/message-id/CA%2BhUKGKFvu3zyvv3aaj5hHs9VtWcjFAmisOwOc7aOZNc5AF3NA%40mail.gmail.com), [#12 (Tom Lane, 2025-11-23)](https://www.postgresql.org/message-id/543920.1763917337%40sss.pgh.pa.us), [#35 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/BGXViuPJAXY5dAefuF31MhArLzQjKYoS60rlNordOBLSX7_pATL66i-2-nYMIHeyoHSGLDQ3plkUjSpW-1SPAE4y0MBdWo8JVFbUj4UKKO4%3D%40burd.me)

**Question:** Which platforms that master builds on today would fail the new lock-free check, and is dropping them an explicit decision?

Counter-check: held. Message #35 says v12 fails configure where 64-bit atomics are not lock-free, and master still emulates those in fallback.h. The proposal does not list which targets lose support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | slock_t and pg_atomic_flag change representation, and pg_atomic_read_u32/u64, which extensions call, get seq_cst ordering. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | v12 removes arch-*.h, generic*.h, fallback.h and s_lock.h, which back branches keep. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v11 keeps two atomics implementations in the tree; v12 depends on compiler codegen and on MSVC's /experimental:c11atomics. |
| performance on the hot path | **yes** | seq_cst reads apply to about 200 pg_atomic_read call sites and are not benchmarked on rv64 or POWER. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which version is proposed for commit, v11 or v12, and if v11, what result and what release would trigger the move to v12?
2. Which caller relies on ordering that pg_atomic_read_u32/u64 does not promise, causing the join_hash tuple loss on rv64 with gcc 13.3?
3. What does the seq_cst read cost on POWER and rv64 under a contended pgbench run?
4. Which CI tasks or buildfarm animals will build the opt-in stdatomic path, and how does a server reject an extension built with the other slock_t layout?
5. Which platforms that master builds today would fail v12's lock-free check, and what test results exist for POWER, s390x and illumos?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Thomas Munro co-wrote the patch, and Tom Lane, Heikki Linnakangas and Peter Eisentraut commented, but none has replied since v11 and v12 were posted. The owner must know atomics, spinlocks and weak-memory architectures.
- **Liability:** The owner signs for the correctness of every atomic and spinlock operation on all supported architectures and compilers, including codegen the project does not control, with no end date.
- **Author follow-up:** Greg Burd posted versions up to v12 and fixed three bugs that a reviewer found in v8 to v10. Thomas Munro has not posted since message #15.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The gain is less in-tree code for maintainers; users get performance parity. The patch touches atomics, spinlocks and the build system on every architecture, and the choice between v11 and v12 and the rv64 tuple loss are open.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` The v11 build option creates two extension ABIs within one major version. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same layout mismatch with more detail (the one-byte x86 slock_t and the "auto" setting).
- `standards-compat-3` C++ extension compatibility of the atomics header is not defined. Refuted (no failure scenario): The commit message says C++11 and later use <atomic>, and message #31 retested the reported extern "C" failure after the fix. The objection names no ABI where lock-free 32- or 64-bit std::atomic and _Atomic differ in layout.
- `already-exists-1` Opt-in v11 adds a second atomics implementation beside port/atomics.h. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about two implementations with no tester and no exit criteria.
- `already-exists-2` Existing gcc and clang path already uses compiler atomic builtins. Refuted (already handled): Message #35 lists what v12 removes: arch-arm.h, arch-ppc.h, arch-x86.h, fallback.h, generic.h, generic-gcc.h, generic-msvc.h, s_lock.h and s_lock_test. Peter also named removal of the in-tree library as a valuable outcome in message #25.
- `demand-1` Default-off second atomics implementation has no named users or test plan. Refuted (not specific): Duplicate of maintenance-fit-1. End-user demand is a weak test for a refactoring whose goal Peter called valuable in message #25.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposes C11 atomics, and neither recorded reason applies here. [P5] reverted a ppc inline-asm change that conflicted with fallback.h, and [P1] said the compiler should own atomics code generation, which this proposal does; message #15 mentions an earlier opt-in version but no rejection reason.
- standards-compat: 3 objection(s). Objections were found; none apply to the SQL surface, pg_dump or psql, which this proposal does not touch.
- replication-ondisk: 1 objection(s). Apart from the ABI point, the proposal touches only in-memory and shared-memory synchronization. It changes no on-disk format, WAL record, catalog or wire protocol, so pg_upgrade and replication are not affected.
- security: 0 objection(s). The proposal changes how atomics and spinlocks are built and adds a build-time option. It adds no SQL object, privilege, role, catalog, monitoring view or connection path, and it does not change what an unprivileged user can do or see. Correctness risks such as the RISC-V tuple loss, weak-memory ordering and untested platforms belong to the safety and design angles. They are not a privilege or exposure issue.
- maintenance-fit: 3 objection(s). Objections raised. The precedents were not cited: P1 is about adding a compiler flag, and P5 is a single ppc revert, so neither recorded reason applies here.
- already-exists: 2 objection(s). Two objections; no contrib module or external tool replaces the in-tree atomics, so no extension-based objection applies.
- demand: 1 objection(s). This is internal infrastructure with a maintainer-side goal that Peter Eisentraut supports in message #25, so v12's code removal has real value to the project. The only demand gap is the opt-in v11 path, which has no stated users.

### Precedents retrieved

- P1: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - used in the review; reason summarised from its thread
- P2: [Atomic pgrename on Windows](https://commitfest.postgresql.org/patch/1386/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P3: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P4: [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (cf, Returned with feedback, 2015-12-16) - judged not relevant; reason summarised from its thread
- P5: [Revert "For all ppc compilers, implement pg_atomic_fetch_add_ with inline asm."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e9fae0696d9e3ff70a1438775ad9f786b854a5) (revert, Reverted, 2019-09-15) - used in the review
- P6: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P7: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant
- P8: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant; reason summarised from its thread
- P9: [Atomic commit support for foreign data wrappers](https://commitfest.postgresql.org/patch/180/) (cf, Returned with feedback, 2016-01-31) - judged not relevant; reason summarised from its thread
- P10: [Additional Chapter "Architecture" for Part "Tutorial"](https://commitfest.postgresql.org/patch/2541/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P11: [Replace spinlock with lwlock in pg_stat_statements](https://commitfest.postgresql.org/patch/2634/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P12: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P13: [remove ISB in AArch64's spin_delay() implementation](https://commitfest.postgresql.org/patch/5763/) (cf, Returned with feedback, 2025-08-19) - judged not relevant
- P14: [Clarify need for memory barriers in bgworkers](https://commitfest.postgresql.org/patch/69/) (cf, Rejected, 2015-02-03) - judged not relevant
- P15: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P16: [Revert "Allow --with-bonjour to work with non-macOS implementations of Bonjour."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d9adab60754ac71b0b500c91c45e12e940b3ce) (revert, Reverted, 2017-11-09) - judged not relevant
- P17: [Allow HAVE_UCOL_STRCOLLUTF8 flag to be set conditionally on Windows](https://commitfest.postgresql.org/patch/1188/) (cf, Rejected, 2017-08-11) - judged not relevant
- P18: [Use internal pthreads reimplementation only when building with MSVC](https://commitfest.postgresql.org/patch/2526/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P19: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P20: [Improve documentation for spinlocks file](https://commitfest.postgresql.org/patch/208/) (cf, Rejected, 2015-07-02) - judged not relevant
- P21: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3534/) (cf, Returned with feedback, 2022-06-29) - judged not relevant
- P22: [Revert removal of signed, volatile, and signal handler arg type tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ef6164de1d60f65039a73f5975d50b4fa10b8a5b) (revert, Reverted, 2000-08-29) - judged not relevant
- P23: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3731/) (cf, Withdrawn, 2022-07-01) - judged not relevant
- P24: [Reloption tests improvement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3747/) (cf, Returned with feedback, 2022-08-02) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0004-Remove-configure-meson-checks-for-atomics.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 167 s; cost reported by the CLI: $3.40
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 98 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:13:24.223Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
