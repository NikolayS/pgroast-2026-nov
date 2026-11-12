# Design pre-review: Fix use-after-free after failed pg_checksum_init

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7257](https://commitfest.postgresql.org/patch/7257/) (Needs review); authors: Yuriy Grigoryev (ju.grigorev)
- Thread: [\[PATCH\] Fix use-after-free after failed pg_checksum_init](https://www.postgresql.org/message-id/flat/9d69505e8d4948048e8401165ca4c3b8%40localhost.localdomain), started 2026-08-17
- Messages read: 6 of 6
- Patch prose read: v3-0001-Fix-UAF-after-failed-pg_checksum_init.patch, v1-0001-Fix-UAF-after-failed-pg_checksum_init.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1 on the linker-substituted test mock, was refuted because both failure modes are loud and the point is test-level. The fix clears a dangling pointer, frees a leaked context, and makes pg_combinebackup exit on checksum errors as basebackup and pg_verifybackup do. A committer agreed with the goal in message #2 and the brief reports v3 as mostly ready.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review could not check that every fallible pg_checksum_init() and pg_checksum_final() call in pg_combinebackup is covered, or that no caller frees the context a second time. Only three of six messages were visible.

## The proposal, restated

**Problem.** pg_checksum_init() frees the SHA context when pg_cryptohash_init() fails but leaves context->raw_context.c_sha2 dangling. pg_combinebackup ignores the -1 return and later calls pg_checksum_update() or pg_checksum_final() with that pointer. The bug affects frontend OpenSSL builds when EVP_DigestInit_ex() fails after a successful create, and pg_checksum_final() also leaks the SHA context when finalization fails.

**Proposed design.** pg_checksum_init() sets the SHA pointer to NULL after freeing it. pg_checksum_final() frees and clears the SHA context after the switch, whether finalization succeeds or fails. pg_combinebackup checks every pg_checksum_init() and pg_checksum_final() result and exits with pg_fatal, as basebackup and pg_verifybackup do.

**User-visible surface.**
- pg_combinebackup exits with an error such as "could not initialize checksum of file" instead of continuing after a checksum init or final failure.
- pg_checksum_final() in src/common now always frees and clears the SHA context, which changes the cleanup contract for frontend callers.
- New uninstalled test program test_checksum_helper and TAP test t/015_checksum_helper.pl in src/bin/pg_combinebackup.

**What the author claims.**
- create_manifest_writer() always uses SHA256, so the bug is reachable without --manifest-checksums=sha256.
- Non-OpenSSL builds do not hit the path, because the in-core cryptohash does not fail after create.
- The success path has no performance or user-visible change.
- The new test fails with master's checksum_helper.c and passes with the patch for all four SHA variants.
- The fix should be back-patched to 17, where pg_combinebackup was added; the author later left this open.

**Thread state.** Daniel Gustafsson calls v3 mostly ready but asks for a clearer if/else with an Assert, a Makefile rule name fix, and one helper call per test case. Raja Sai pranav tested v2 and found it correct; back-patching is still undecided.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pg_checksum_final() in src/common now always frees and clears the SHA context, which changes the cleanup contract for callers. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 asks for a back-patch to 17; message #2 calls the risk/reward unfavorable, and the question is still open. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds test_checksum_helper, a test program that links checksum_helper.c against mock pg_cryptohash_* functions. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is there a v4 that answers the review comments of 2026-11-06 on the if/else with an Assert, the Makefile rule name, and one helper call per test case?
2. Should the fix go to 17 and later, and if so, does the back-branch version include the new test program or only the code change?
3. Does test_checksum_helper link and run with the mock cryptohash functions on Windows and macOS, under both meson and autoconf?
4. Does any in-tree or known out-of-tree caller of pg_checksum_final() use the SHA context after the call and so depend on the old cleanup behaviour?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson reviewed the patch in message #2 and, per the brief, calls v3 mostly ready. The owner needs to know the frontend cryptohash and OpenSSL code and the backup tools.
- **Liability:** The owner signs for the changed cleanup contract of pg_checksum_final() in src/common, for the new mock-linked test program on all buildfarm platforms, and for any back-branch destabilization if the fix goes to 17.
- **Author follow-up:** The author answered review within a day in message #3 and, per the brief, posted up to v3 with a reworked function and a new test. The thread shows no v4 yet for the latest comments.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The error path needs an OpenSSL init or final failure, which message #2 calls unlikely, so few users will ever hit it. The v3 patch is about 16 KiB over nine files in pg_combinebackup and one src/common file, and a committer is already reviewing it.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Linker-substituted cryptohash mock is a new, fragile test mechanism. Refuted (no failure scenario): Both outcomes in the scenario are loud: a link error or a failing test, so item 9 does not fail. Daniel Gustafsson reviewed the v3 test and asked only for small changes; untested Windows linking is a test-level question outside levels 1-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The one retrieved row, [P11], reverted a 2020 switch to OpenSSL EVP routines because the wrappers raised errors instead of returning a status. This patch adds the status checks that design asked for, so the recorded reason does not apply. Neither the thread nor the patch prose points to an earlier discussion of this fix.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to: the patch adds no syntax, GUC, catalog object, psql behaviour or dump format, and pg_combinebackup now stops with pg_fatal, as basebackup and pg_verifybackup already do (message #1, #2). The pg_checksum_final() change frees and clears a SHA context that callers could not free before, so an out-of-tree caller that touches the pointer afterwards sees NULL, not freed memory. [P11] is about backend EVP error handling and does not apply here.
- replication-ondisk: 0 objection(s). This angle finds no objection, because the patch changes no on-disk format, WAL record, catalog, wire protocol or struct layout. pg_combinebackup's output, including backup_manifest, is unchanged on success, and on failure the program now exits with pg_fatal. The pg_checksum_final() change leaves pg_checksum_context unchanged and only frees and clears the SHA pointer on the failure path, so a later pg_cryptohash_free() on the cleared pointer should do nothing. [P11] concerns backend OpenSSL wrapper design and does not apply here.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pg_combinebackup is a frontend tool run by the backup operator on files they already own, and the bug needs an OpenSSL init failure that an unprivileged database user cannot trigger. The fix adds error exits and pointer clearing, with no new access, data exposure or resource use. Back-patching is a stability question, which other angles cover.
- maintenance-fit: 1 objection(s). The fix itself is small and matches basebackup and pg_verifybackup, and a committer (message #2, #6) is engaged. Only the test harness carries long-run cost. P11 is about EVP wrapper design and does not apply.
- already-exists: 0 objection(s). This is a bug fix for a dangling pointer and a leak in src/common/checksum_helper.c, plus missing return-value checks in pg_combinebackup. No existing feature, option or extension covers it. The new mock test harness is not a second injection-point mechanism, because injection points are backend-only and raise errors instead of returning -1 (message #3), and the harness follows the test_escape pattern. [P11] concerns the OpenSSL EVP wrapper design, not this fix.
- demand: 0 objection(s). This is a memory-safety fix, so it needs no user demand to justify it. A committer (message #2, #6) agreed that pg_combinebackup should fail like basebackup does, and the cost falls only on the error path. Back-patch value and the new test program's upkeep belong to other angles.

### Precedents retrieved

- P1: [Checksums for slru files](https://commitfest.postgresql.org/patch/1462/) (cf, Returned with feedback, 2018-03-02) - judged not relevant; reason summarised from its thread
- P2: [Online checksums verification](https://commitfest.postgresql.org/patch/2369/) (cf, Returned with feedback, 2020-11-05) - judged not relevant
- P3: [Online verification of checksums](https://commitfest.postgresql.org/patch/1733/) (cf, Returned with feedback, 2019-04-09) - judged not relevant
- P4: [SLRU data page checksums](https://commitfest.postgresql.org/patch/1682/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P5: [Refactoring of connection with password prompt loop for frontends](https://commitfest.postgresql.org/patch/2248/) (cf, Returned with feedback, 2019-09-27) - judged not relevant; reason summarised from its thread
- P6: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P7: [pg_checksum: add test for coverage](https://commitfest.postgresql.org/patch/3852/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [teach pg_combinebackup to detect deleted files where possible](https://commitfest.postgresql.org/patch/4951/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P9: [change SOFT_ERROR_OCCURRED to a static inline function](https://commitfest.postgresql.org/patch/6666/) (cf, Withdrawn, 2026-08-07) - judged not relevant; reason summarised from its thread
- P10: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P11: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - used in the review
- P12: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P13: [Mingw: Fix import library extension, build actual static libraries](https://commitfest.postgresql.org/patch/2082/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P14: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P15: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant
- P16: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P17: [pg_upgrade: fix memory leak in SLRU I/O code](https://commitfest.postgresql.org/patch/6462/) (cf, Withdrawn, 2026-02-05) - judged not relevant
- P18: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant
- P19: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P20: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P21: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P22: [Logical Replication Helper Functions](https://commitfest.postgresql.org/patch/45/) (cf, Rejected, 2015-02-16) - judged not relevant
- P23: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant
- P24: [OpenSSL 3.0.0 compatibility](https://commitfest.postgresql.org/patch/2810/) (cf, Withdrawn, 2021-03-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 105 s; cost reported by the CLI: $1.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:33:35.893Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
