# Design pre-review: Let's drop support for named POSIX semaphores

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7513](https://commitfest.postgresql.org/patch/7513/) (Needs review); authors: Tom Lane (tgl)
- Thread: [Let's drop support for named POSIX semaphores](https://www.postgresql.org/message-id/flat/507767.1792981204%40sss.pgh.pa.us), started 2026-10-26
- Messages read: 11 of 11
- Patch prose read: v1-0001-Drop-support-for-named-POSIX-semaphores-the-sem_o.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The proposal states the problem and the change, removes an untested path that meson cannot select, and leaves SysV and unnamed POSIX semaphores in place. The one open point is whether an out-of-tree configure build still selects named semaphores; the material shows none.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review cannot confirm that the configure and darwin template changes leave SysV and unnamed POSIX selection unchanged on every platform. Only 3 of 11 messages were shown.

## The proposal, restated

**Problem.** Postgres keeps a sem_open() semaphore implementation that no platform prefers and that nobody tests. macOS, the original reason for it, has used SysV semaphores for a long time, and the meson build offers no way to select it.

**Proposed design.** The patch removes the named POSIX semaphore path from posix_sema.c and the configure, meson and darwin template logic that can select it. SysV semaphores and unnamed POSIX semaphores (sem_init) stay as the two supported implementations.

**User-visible surface.**
- The configure-time choice of named POSIX semaphores (USE_NAMED_POSIX_SEMAPHORES in pg_config.h.in) goes away.

**What the author claims.**
- Named POSIX semaphores do not work with EXEC_BACKEND.
- They typically use a file descriptor or similar resource per semaphore per backend, so the cost grows as N squared for N backends.
- No platform prefers them, and the OpenBSD tests showed poor performance that any platform would share because of the POSIX API.
- The code is long untested, and meson cannot select it today.

**Thread state.** Shihao Zhong, Heikki Linnakangas and Greg Burd support it, and Peter Eisentraut withdrew his concern about adding a SysV dependency for macOS sandboxing. The discussion moved to a futex-based replacement for semaphores, and Tom plans follow-up fixes for the unused IPCProtection macro, runtime.sgml and comments.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch removes USE_NAMED_POSIX_SEMAPHORES from pg_config.h.in, a macro that out-of-tree code could test. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch deletes a semaphore path in posix_sema.c and its configure, meson and darwin template logic, which lowers the burden. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does any packager or out-of-tree build still select named POSIX semaphores through configure, for example by setting USE_NAMED_POSIX_SEMAPHORES?
2. Will the release notes mention the removal so that anyone who builds with named semaphores learns of it?
3. Will a v2 include the cleanup from message #2: the unused IPCProtection macro, the unneeded includes, and the runtime.sgml sentence?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane, a committer, wrote the patch, and Heikki Linnakangas supported it in message #3. The owner needs to know the port-level semaphore code and the configure and meson build logic.
- **Liability:** The owner signs for build breakage on any platform or out-of-tree build that still selected sem_open(). The risk is small and shows up at build time, soon after commit.
- **Author follow-up:** The thread state says the author plans follow-up fixes for the IPCProtection macro, runtime.sgml and comments.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small, touches six files, and only removes code that nobody tests and meson cannot select. A committer wrote it, three people support it, and the one concern raised in the thread was withdrawn.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found that proposed or rejected removing named POSIX semaphores. [P10] reverted removal of a macOS readline workaround because macOS still ships old libedit; here macOS has preferred SysV semaphores since 2002 and meson cannot select named semaphores, so that reason does not apply. [P5] supports the removal, since one-FD-per-semaphore platforms broke a test. Message #1 cites an OpenBSD discussion that found named semaphores perform poorly and did not proceed.
- standards-compat: 0 objection(s). The only user-facing contract is the configure-time selection of named semaphores. The meson build already cannot select it, and macOS has defaulted to SysV for years, so no build, script or default behaviour should change. No SQL, GUC, catalog, pg_dump or psql surface is touched. [P10] reverted a macOS-related removal, but its reason was an old libedit still shipped in the base system, and the proposal names no platform that still relies on sem_open(). A release-note line for packagers is a small follow-up, not an objection.
- replication-ondisk: 0 objection(s). The patch changes no persistent or cross-node state: no on-disk format, WAL, catalog, protocol or replication change. Named semaphores live only while the server runs, and removing the pg_config.h macro is a major-release build change with no stated extension ABI impact. Platform portability, such as [P10], belongs to other angles.
- security: 0 objection(s). The patch removes a build-time semaphore option and adds no privilege, exposure or authentication surface. If anything, it removes named semaphores that live in a system-wide namespace, which shrinks the exposure. The remaining SysV and unnamed POSIX paths are unchanged, so no security or privilege objection applies.
- maintenance-fit: 0 objection(s). The patch removes code, so long-run maintenance cost goes down: one fewer configure branch and one fewer path in posix_sema.c that nobody tests and meson cannot build. It adds nothing on any hot path, and it reverts the darwin template only where it already prefers SysV semaphores. The patch author is a committer who knows the area, and Heikki Linnakangas also supports it. The only remaining concern is the macOS sandboxing direction, which Peter Eisentraut withdrew in message #11. The one leftover risk is an out-of-tree build that sets USE_NAMED_POSIX_SEMAPHORES through configure, but the material shows no such user. [P10], a macOS-related removal that was later reverted, involved a library macOS still ships, so its reason does not apply here.
- already-exists: 0 objection(s). This angle does not apply to a removal. The proposal deletes a second sem_open() path while SysV and unnamed POSIX semaphores stay, which reduces duplicated mechanisms. No extension or existing feature competes with it, and the futex direction discussed in messages #5, #8 and #10 is a separate future proposal.
- demand: 0 objection(s). The proposal removes code and adds no feature, so the demand question applies to the code being removed. The material shows no current user of named POSIX semaphores. macOS has preferred SysV semaphores since 2002, meson cannot select the named kind, and the OpenBSD discussion in message #1 ended on poor performance. Four people supported the removal in the thread, and Peter Eisentraut withdrew his concern in message #11. [P10] was a removal reverted because of an old macOS libedit, but that reason does not apply here because macOS has had SysV semaphores for decades. The material cannot show whether any out-of-tree configure build still selects USE_NAMED_POSIX_SEMAPHORES; a release-note mention would cover that case.

### Precedents retrieved

- P1: [Revert "ICU: do not convert locale 'C' to 'en-US-u-va-posix'."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=455f948b0d03a556533a7e4a1a8abf45f0eb202e) (revert, Reverted, 2023-05-09) - judged not relevant
- P2: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P3: [Revert WAL posix_fallocate() patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b1892aaeaaf34d8d1637221fc1cbda82ac3fcd71) (revert, Reverted, 2013-09-05) - judged not relevant
- P4: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P5: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - used in the review
- P6: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P7: [Named Operators](https://commitfest.postgresql.org/patch/4123/) (cf, Returned with feedback, 2023-07-04) - judged not relevant; reason summarised from its thread
- P8: [some namespace.c refactoring](https://commitfest.postgresql.org/patch/4212/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P9: [Revert "Fix bogus %name-prefix option syntax in all our Bison files."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71ed8b3ca76cd720f4013c3c20f0d2706583ab9a) (revert, Reverted, 2014-05-28) - judged not relevant
- P10: [Revert "Remove old readline API workaround"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39c636a1aac8fe14afd3dd5af0141dc532a297b7) (revert, Reverted, 2026-08-25) - used in the review
- P11: [Memoize ANTI and SEMI JOIN inner](https://commitfest.postgresql.org/patch/5636/) (cf, Withdrawn, 2025-06-30) - judged not relevant; reason summarised from its thread
- P12: [rand48 replacement](https://commitfest.postgresql.org/patch/3132/) (cf, Withdrawn, 2021-05-24) - judged not relevant; reason summarised from its thread
- P13: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P14: [Plan a filtering inner join as a semijoin](https://commitfest.postgresql.org/patch/7202/) (cf, Withdrawn, 2026-08-27) - judged not relevant
- P15: [Revert "Allow --with-bonjour to work with non-macOS implementations of Bonjour."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d9adab60754ac71b0b500c91c45e12e940b3ce) (revert, Reverted, 2017-11-09) - judged not relevant
- P16: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - judged not relevant
- P17: [Close backend file handles on exec](https://commitfest.postgresql.org/patch/886/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P18: [Possible replace of strncpy on xactdesc.c](https://commitfest.postgresql.org/patch/6989/) (cf, Rejected, 2026-08-21) - judged not relevant
- P19: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - judged not relevant
- P20: [Replace "implementor" with "implementer"](https://commitfest.postgresql.org/patch/3957/) (cf, Withdrawn, 2022-10-23) - judged not relevant
- P21: [Revert workarounds for -Wmissing-braces false positives on old GCC](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=618c64ffd3967cb5313b4b11e1e1043a074f2139) (revert, Reverted, 2025-03-20) - judged not relevant
- P22: [WIP: macOS SIP workaround](https://commitfest.postgresql.org/patch/2862/) (cf, Returned with feedback, 2021-03-08) - judged not relevant
- P23: [\[PATCH\] Make NumericVar storage semantics explicit](https://commitfest.postgresql.org/patch/6751/) (cf, Withdrawn, 2026-07-26) - judged not relevant
- P24: [Have BackendXidGetPid return pid_t](https://commitfest.postgresql.org/patch/6229/) (cf, Rejected, 2026-01-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 76 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 41 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:17.359Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
