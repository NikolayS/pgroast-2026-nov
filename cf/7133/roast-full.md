# Design pre-review: Fix pg_basebackup long-path test on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7133](https://commitfest.postgresql.org/patch/7133/) (Needs review); authors: Harrison Booth (hbooth)
- Thread: [\[PATCH v1\] Fix pg_basebackup long-path test on Windows](https://www.postgresql.org/message-id/flat/CAAH-eSV4BJS7W%3DqRf7%3DHk09XD3NhFOVvvKRjH%2BLh0v9e_S%3DSeg%40mail.gmail.com), started 2026-07-26
- Messages read: 4 of 4
- Patch prose read: v2-0001-Fix-pg_basebackup-long-path-test-on-Windows.patch, 0001-Fix-pg_basebackup-long-path-test-on-Windows.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case in message #1: a 253-character path, an unchecked mkdir, and a misleading later error. The fix changes one TAP test file and has no user-visible surface. Confidence is medium because the code was not read and the v3 diff is not shown.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and the v3 diff were not read. Two things could not be checked: whether the cleanup warnings come from this change, and whether the test still covers a long tablespace path.

## The proposal, restated

**Problem.** On Windows, the pg_basebackup TAP test creates $superlongname through the build-tree side of a junction, so the path can exceed the 248-character directory limit. Perl's mkdir then fails without a check, and the test later fails with a misleading CREATE TABLESPACE "directory does not exist" error. This affects developers and buildfarm members running the tests on Windows.

**Proposed design.** The patch creates the long-named tablespace directory through the short system temporary directory side of the junction. It also checks the result of mkdir and calls BAIL_OUT with the `$!` variable. on failure, so a setup failure is reported where it happens. Only src/bin/pg_basebackup/t/010_pg_basebackup.pl changes.

**What the author claims.**
- The unpatched test fails and the patched test passes all 149 subtests on native ARM64 and on an x64 cross-build, Windows 11, MSVC 19.44.
- v3, rebased on 8e82d6610e7, passes all 151 subtests on native ARM64 and x64 under ARM64 emulation, with directory-cleanup warnings.
- A GitHub-hosted Windows x64 build passes all 152 pg_basebackup subtests, with "all three patches" applied.
- No earlier proposal covers this unchecked long-directory creation; a 2023 thread covered a related long-file test.

**Thread state.** Daniel Gustafsson asked for the `$!` variable. in the BAIL_OUT message, and v2 added it. The author posted v3 as a rebase and asked for another look, but nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix for a test in 010_pg_basebackup.pl, and the thread does not say which stable branches need it. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the test fail on any buildfarm member or on CI today, or only when the build-tree path is long enough to pass 248 characters?
2. Do the directory-cleanup warnings in the v3 runs also appear without the patch, or does creating the directory through the short side of the junction cause them?
3. Should the fix be back-patched, and if so, to which stable branches does the same unchecked mkdir apply?
4. What are the "three patches" applied in the GitHub-hosted Windows x64 run, given that the thread attaches only one?
5. Does the test still exercise a long tablespace path after the directory is created through the short system temporary directory?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson commented on v1 in message #2, and the demand review note describes him as a committer. Nobody has replied to v3 yet. The owner needs to know the TAP test framework and Windows path handling.
- **Liability:** One TAP test file. The owner signs for the test passing on Windows buildfarm members and for any cleanup warnings the new directory location may cause. The liability is small and limited to the test suite.
- **Author follow-up:** The author posted v2 in response to review and rebased to v3 with new test runs, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 1.5 KB in one test file and helps developers and buildfarm members who run the suite on Windows with long build paths. It adds no runtime code and makes a setup failure report at its source.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the corpus search returned no related rejected attempt. Message #1 points to a 2023 discussion of a related pg_basebackup long-file test, but the thread does not report it as a rejection of this fix, so no earlier objection applies.
- standards-compat: 0 objection(s). The patch changes only one TAP test file and has no user-facing surface: no SQL syntax, GUC, catalog, pg_dump output, psql behaviour or client-visible change, so the standards and compatibility angle finds nothing to object to. Whether the test still covers long tablespace paths when the directory is created through the short side of the junction is a test-coverage question for another angle.
- replication-ondisk: 0 objection(s). The patch changes only one TAP test script, src/bin/pg_basebackup/t/010_pg_basebackup.pl. It does not touch on-disk format, WAL, replication, catalogs, the wire protocol or the extension ABI, so this angle has nothing to object to.
- security: 0 objection(s). The patch changes only a TAP test script, 010_pg_basebackup.pl, and makes no change to server, client, privileges, authentication or anything a user can reach. The temporary directory and junction already exist in the test, so the security angle has nothing to object to.
- maintenance-fit: 0 objection(s). The patch changes one TAP test file, adds no runtime code, no API and no hot-path cost, and the mkdir check makes future setup failures loud. A committer may ask whether the directory-cleanup warnings seen in the v3 runs come from this change, but that is a question for review, not an objection.
- already-exists: 0 objection(s). This is a test-only fix in 010_pg_basebackup.pl, and I know of no existing feature, extension or tool that already solves it. The patch reuses the short-path junction that the test already creates, so it adds no second mechanism and leaves nothing for an extension or hook to do.
- demand: 0 objection(s). This is a test-only fix for a reproducible failure on Windows builds, and it costs nothing outside one TAP file. Demand is not the right test here. Developers and buildfarm members who run the suite on Windows need a test that passes or fails where the real error is, and a committer reviewed it in message #2.

### Precedents retrieved

- P1: [pg_basebackup: add test about zstd compress option](https://commitfest.postgresql.org/patch/3835/) (cf, Rejected, 2022-12-06) - judged not relevant; reason summarised from its thread
- P2: [Prevent pg_basebackup running as root](https://commitfest.postgresql.org/patch/2431/) (cf, Rejected, 2020-02-27) - judged not relevant; reason summarised from its thread
- P3: [Application name for pg_basebackup, pg_receivewal and pg_recvlogical](https://commitfest.postgresql.org/patch/2335/) (cf, Rejected, 2019-11-07) - judged not relevant; reason summarised from its thread
- P4: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant; reason summarised from its thread
- P5: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant; reason summarised from its thread
- P6: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P7: [pg_basebackup has stricter tablespace rules than the server](https://commitfest.postgresql.org/patch/1125/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P8: [Revert "Prevent running pg_basebackup as root"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcddc3f813ff39d0e33c2b7c77092a6c0b0e9d0b) (revert, Reverted, 2020-02-07) - judged not relevant
- P9: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P10: [Revert changes to pg_basebackup and pg_waldump usage() code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05b5feb60e7cf36db08eb0d34dbd21854cb9e8ce) (revert, Reverted, 2017-05-17) - judged not relevant
- P11: [Add tablespace tap test to pg_rewind](https://commitfest.postgresql.org/patch/2059/) (cf, Returned with feedback, 2019-05-10) - judged not relevant; reason summarised from its thread
- P12: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant; reason summarised from its thread
- P13: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant
- P14: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [Revert "Switch TAP tests of pg_rewind to use a role with minimal permissions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=db8db624e826efbe16aab1ae921bae071f98f099) (revert, Reverted, 2019-04-13) - judged not relevant
- P16: [Rewriting the test of pg_upgrade as a TAP test - take three](https://commitfest.postgresql.org/patch/3120/) (cf, Withdrawn, 2021-11-18) - judged not relevant
- P17: [Rewriting the test of pg_upgrade as a TAP test - take two](https://commitfest.postgresql.org/patch/1497/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P18: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant
- P19: [List TAP test files in makefiles](https://commitfest.postgresql.org/patch/6011/) (cf, Rejected, 2025-12-03) - judged not relevant
- P20: [Revert "Fix link error for test_radixtree module on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6d9751fa8fd40c988541c9c72ac7a2095ba73c19) (revert, Reverted, 2024-03-08) - judged not relevant
- P21: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P22: [Revert "Prepare for Python "Limited API" in PL/Python"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f734c9fc3a91959c2473a1e33fd9b60116902175) (revert, Reverted, 2025-02-26) - judged not relevant
- P23: [TAP test module - PostgresClient](https://commitfest.postgresql.org/patch/1442/) (cf, Rejected, 2018-03-03) - judged not relevant
- P24: [Revert "Add new simple TAP test for tablespaces."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c53a6658e47217ad3dd416a5543fc87c3ecfd58) (revert, Reverted, 2022-01-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 81 s; cost reported by the CLI: $0.59
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 3 finding(s), 3 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:23:02.281Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
