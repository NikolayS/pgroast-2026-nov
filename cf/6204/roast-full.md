# Design pre-review: Windows relation extension failure at 2GB and 4GB

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6204](https://commitfest.postgresql.org/patch/6204/) (Needs review); authors: Bryan Green (lispyone)
- Thread: [\[Patch\] Windows relation extension failure at 2GB and 4GB](https://www.postgresql.org/message-id/flat/0f238ff4-c442-42f5-adb8-01b762c94ca1%40gmail.com), started 2025-10-28
- Messages read: 32 of 32
- Patch prose read: v6-0001-Fix-Windows-file-I-O-for-offsets-beyond-2GB.patch, 0001-Fix-Windows-file-IO.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The I/O fix is committed to HEAD and no one disputes the goal. What remains is the test module and the v6 meson.build change, and all three surviving objections are fixable level 2 points. standards-compat-1 is the strongest: v6 removes the meson segment size check while other code still uses off_t. maintenance-fit-2 and maintenance-fit-3 repeat requests from the engaged committer that v6 does not yet meet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. What v6 changes in meson.build is known only from the quote in message #28, and how the test behaves on a filesystem without sparse files could not be checked.

## The proposal, restated

**Problem.** On Windows with MSVC, off_t is 32 bits, so relation files built with a segment size above 1GB fail to extend at 2GB with "could not extend file: Invalid argument". Past the 2GB fix, the Windows pg_pwrite()/pg_pread() wrap at 4GB because they never set OffsetHigh. This affects only people who build with a larger segment size, which the author says is rare.

**Proposed design.** The patch replaces off_t with pgoff_t in fd.c, md.c, pg_iovec.h, file_utils.c and their headers, and sets both Offset and OffsetHigh in the Windows OVERLAPPED wrappers. v6 adds a test module, test_large_files, which writes and reads a sparse temporary file at 4GB+1 through the VFD layer and checks FileSize(). v6 also touches the top-level meson.build.

**User-visible surface.**
- C API: File* functions in fd.h such as FileSize() and FileTruncate() take or return pgoff_t instead of off_t.
- C API: pg_preadv(), pg_pwritev(), pg_pread(), pg_pwrite() and pg_pwrite_zeros() take pgoff_t offsets.
- New test module test_large_files with SQL functions test_large_files_offset_size() and test_large_files_test_4gb_boundary().
- Build: meson check that rejects a segment size of 2GB or more when off_t is under 8 bytes, committed and backpatched to v16 separately.

**What the author claims.**
- Both bugs reproduce reliably with segsize=8, failing at exactly 2GB and then at exactly 4GB.
- The change is safe on Unix because pgoff_t equals off_t there, so only Windows behavior changes.
- The test at 4GB+1 catches both bugs, while a test at 2GB+1 would miss the OffsetHigh bug.
- The test completes in seconds on most platforms.

**Thread state.** Michael Paquier committed the meson segment size check down to v16 and the I/O routine changes to HEAD; the open part is the test module. In the last message, Michael asks for modular SQL wrappers around File* functions and a .gitignore, questions the pgoff_t size check, and opposes dropping the meson check while buffile.c still uses off_t.

## 2. Prior attempts

### Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes) (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/5403/](https://commitfest.postgresql.org/patch/5403/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKZiRmyM4YnokK6Oenw5JKwAQ3rhP0YTz2T-tiw5dAQjGRXE3Q@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKZiRmyM4YnokK6Oenw5JKwAQ3rhP0YTz2T-tiw5dAQjGRXE3Q@mail.gmail.com)
- **Date:** decided 2024-11-25; started 2024-11-21
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The last message discusses how to back-patch the fix, including an ftruncate() replacement, and nobody objects to it. The entry was withdrawn three days later, and no message gives a reason.
- **Reason (quote):** quote unavailable
- **What is different this time:** P1 fixed 32-bit off_t in frontend tar writing (lseek, ftruncate) for pg_basebackup. This proposal fixes the backend fd.c and md.c layer and the OVERLAPPED wrappers. P1 was withdrawn with no recorded reason, so no objection carries over.

### Large files for relations (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4305/](https://commitfest.postgresql.org/patch/4305/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKG+BGXwMbrvzXAjL8VMGf25y_ga_XnO741g10y0=m6dDiA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKG+BGXwMbrvzXAjL8VMGf25y_ga_XnO741g10y0=m6dDiA@mail.gmail.com)
- **Date:** decided 2024-10-07; started 2023-05-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut, Pavel Stehule (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author dropped the first plan because segmented mode cannot be removed: ext4 limits files to 16TB, and whole-relation files make file-level backups coarser. The --rel-segsize patch then missed v17 over open problems with incremental backup: pg_combinebasebackup and the segment size, and memory use in basebackup_incremental.c that grows with segment size. Peter Eisentraut's May 2024 review found that the patch needed a rebase and that pgbench/001_pgbench_with_server fails with a 64kB segment size. No revised patch appears in the visible messages.
- **Reason (quote):** "If I run it with 64kB, the test pgbench/001_pgbench_with_server fails consistently, so it seems there is still a gap somewhere." ([message](https://www.postgresql.org/message-id/9b0235f1-f1f1-47d8-a764-43ea06fdb65f%40eisentraut.org))
- **What is different this time:** P2 aimed at a per-cluster segment size set at initdb and stalled on incremental backup. This proposal keeps the compile-time segment size and only makes offsets 64-bit on Windows, so P2's recorded problems do not apply.


## 3. Objections a reviewer is likely to raise

### 1. v6 removes the meson segment size check before all relation-file paths are converted

fixable · level 2, rubric item 13 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A packager builds with MSVC and -Dsegsize=4, which v6 lets meson accept. The server extends relation files past 2GB, but CREATE DATABASE with file copy, pg_rewind or pg_basebackup handles those files through code that still uses off_t. The failure shows up later, during a copy, rewind or backup.

Message #28 quotes v6 removing the `cc.sizeof('off_t') < 8` check from meson.build. The commit message does not mention this change. The thread names off_t users that remain: buffile.c (message #30), and copydir.c and pg_rewind's libpq_source.c and filemap.c (message #2). configure keeps its own check, so after v6 the two build systems would accept different segment sizes for the same compiler.

Thread: [#2 (Michael Paquier, 2025-11-06)](https://www.postgresql.org/message-id/aQwsi2IfG0imIwe-%40paquier.xyz), [#28 (Michael Paquier, 2025-12-02)](https://www.postgresql.org/message-id/aS5TIG9BIKrTrZz4%40paquier.xyz), [#30 (Michael Paquier, 2025-12-02)](https://www.postgresql.org/message-id/aS5WzzmLvlXuKbQp%40paquier.xyz)

**Question:** Which code paths that read, copy or back up relation segment files still use off_t, and can the meson check stay until all of them use pgoff_t?

Counter-check: held. Message #28 quotes the v6 hunk that removes the check, and messages #2 and #30 name off_t users that remain. No visible reply from the author addresses this, and keeping the check is a small change.

### 2. A 4GB sparse-file test runs in every check with no opt-in gate

fixable · level 2, rubric item 4 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A buildfarm animal or developer runs make check on a filesystem without sparse file support, or on a small SD card. The test writes past 4GB, so the run fills the disk or becomes very slow. Everyone who runs check pays this cost, but only MSVC builds with a large segment size gain anything.

In message #24, Michael proposed a new PG_TEST_EXTRA value for this reason. The v6 README mentions only NO_INSTALLCHECK and claims the test completes in seconds "on most platforms". Sparse files are a property of the filesystem, and nothing checks for them before the test writes past 4GB. Both the README and message #26 frame the gate as optional.

Thread: [#24 (Michael Paquier, 2025-11-13)](https://www.postgresql.org/message-id/aRVZDTFwNwBUdyGe%40paquier.xyz), [#26 (Michael Paquier, 2025-11-14)](https://www.postgresql.org/message-id/aRbP4uSwd0Edpp5S%40paquier.xyz), [#27 (Bryan Green, 2025-11-27)](https://www.postgresql.org/message-id/3902ccf2-cf32-4f35-9925-2240e34c9141%40gmail.com)

**Question:** What happens to make check on a filesystem without sparse files, and why should the test not be gated by PG_TEST_EXTRA?

Counter-check: held. Message #24 asks for a PG_TEST_EXTRA gate and v6 sets only NO_INSTALLCHECK. Message #27 dropped FSCTL_SET_SPARSE, so the write at 4GB+1 may allocate real disk space on NTFS and on filesystems without sparse files.

### 3. Test module is a single fixed scenario, not reusable File* wrappers

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

The off_t conversion of buffile.c or pg_rewind later needs a test of its own. The hard-coded test_large_files_test_4gb_boundary() cannot express it, so a second module or another fixed C function gets added. The tree then holds several narrow I/O test modules for one problem.

Michael asked in message #9 and again in message #28 for thin SQL wrappers around File* functions that take bytea and an offset. The v6 test is one C function that runs a fixed sequence. It also checks sizeof(pgoff_t), which is 8 bytes in every build (message #28), so that check can never skip anything. No later version shows the requested design.

Thread: [#9 (Michael Paquier, 2025-11-06)](https://www.postgresql.org/message-id/aQ0djavBB5G4QMbS%40paquier.xyz), [#28 (Michael Paquier, 2025-12-02)](https://www.postgresql.org/message-id/aS5TIG9BIKrTrZz4%40paquier.xyz)

**Question:** Can the module expose open, write, read and size wrappers that take an offset and bytea, so that later off_t conversions reuse it?

Counter-check: held. The test module is the open part of the patch, and the engaged committer asks for modular File* wrappers in message #28, with no visible reply. The sizeof(pgoff_t) check is always true, as message #28 says.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | File* functions in fd.h and pg_pread(), pg_pwrite(), pg_preadv(), pg_pwritev() change their offset type from off_t to pgoff_t. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The signature changes went to HEAD only because of ABI risk; only the meson segment size check was backpatched to v16. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new test module, test_large_files, and an off_t conversion that is only partly done across the tree. |
| performance on the hot path | **yes** | The patch changes offset types in fd.c, md.c and the pread/pwrite wrappers, which every relation read and write uses. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v6 remove the meson check that rejects a segment size of 2GB or more when off_t is under 8 bytes, and if so, why?
2. Which code paths that read, copy or back up relation segment files still use off_t, and when will they move to pgoff_t?
3. What happens to make check on a filesystem without sparse files, and why is the test not gated by a PG_TEST_EXTRA value?
4. Can the module expose open, write, read and size wrappers that take an offset and bytea, as message #28 asks?
5. Why does the v6 commit message describe the I/O fix when its file list holds only meson.build and the test module?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed the meson check and the I/O changes and reviews the test module in message #28. The owner needs to know the Windows port and the fd.c layer.
- **Liability:** The owner signs for a test module that writes past 4GB on every buildfarm animal, and for the meson check staying correct until the remaining off_t users are converted.
- **Author follow-up:** The author posted six versions and answered review within a day early in the thread (message #3). No version after v6 and no reply to message #28 is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix helps only Windows MSVC builds with a segment size above 1GB, but it is already committed and the rest is a small test module. A committer is engaged and has listed concrete changes, so a v7 that answers message #28 can move forward.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Dropping the meson segment size check exposes pg_rewind and backup paths. Refuted (not specific): Duplicate of standards-compat-1. The claim that pg_rewind writes wrong data without an error goes beyond message #2, which says only that it "seems also broken".
- `maintenance-fit-1` v6 drops the meson segment size check while buffile.c still uses off_t. Refuted (not specific): Duplicate of standards-compat-1, which covers the same removed check and the same remaining off_t users.

### Review angles

- prior-attempts: 0 objection(s). Neither precedent was stopped by a reason that applies to this proposal. P1 has no recorded reason, and P2 stalled on runtime segment size and incremental backup. In message #4, Thomas Munro says P2 moved from spreading pgoff_t to blocking oversized segments. That block is now committed separately (message #19), and the pgoff_t change went to HEAD only (message #24).
- standards-compat: 1 objection(s). The C API change to pgoff_t is HEAD only and was committed without backpatching, which avoids a minor-release ABI break. The test module adds no user-facing SQL or GUC surface.
- replication-ondisk: 1 objection(s). Apart from the meson check, nothing persistent changes. The on-disk format, WAL, catalogs and the wire protocol stay the same, and the fd.h ABI change is limited to HEAD, as agreed in message #2.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes internal offset types and the Windows OVERLAPPED wrappers, and it adds a test module that is not installed by default and is marked NO_INSTALLCHECK. The planned File* SQL wrappers in message #28 would accept raw File numbers, but this is acceptable only while they stay in a test module that is never installed in production. Disk use without sparse files is a test infrastructure concern for PG_TEST_EXTRA, not a security one.
- maintenance-fit: 3 objection(s). The I/O type change itself is committed to HEAD and was supported by Michael, Thomas and Andres, so no objection is raised against it.
- already-exists: 0 objection(s). No existing feature gives Windows MSVC builds correct 64-bit offsets in fd.c, md.c and the OVERLAPPED wrappers; the default 1GB segment size only avoids the bug. A committer already pushed the I/O changes. The remaining test module does not duplicate any existing test module or extension that I can name with confidence.
- demand: 0 objection(s). The demand objection does not hold. The I/O fix is committed to HEAD, and Paquier, Munro and Freund all asked for the pgoff_t cleanup and the test module (messages #2, #4, #8, #9). The remaining cost, the 4GB test on small machines, is a design point Paquier already raised (message #24).

### Precedents retrieved

- P1: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - used in the review; reason summarised from its thread
- P2: [Large files for relations](https://commitfest.postgresql.org/patch/4305/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P3: [always use runtime checks for CRC-32C instructions](https://commitfest.postgresql.org/patch/4634/) (cf, Withdrawn, 2023-10-31) - judged not relevant; reason summarised from its thread
- P4: [Windows filesystem support improvements](https://commitfest.postgresql.org/patch/3951/) (cf, Returned with feedback, 2023-03-03) - judged not relevant; reason summarised from its thread
- P5: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P6: [invalid data in file backup_label problem on windows](https://commitfest.postgresql.org/patch/2938/) (cf, Withdrawn, 2021-04-06) - judged not relevant; reason summarised from its thread
- P7: [pgrename_temp() – concurrent rename of temporary files on Windows](https://commitfest.postgresql.org/patch/2230/) (cf, Returned with feedback, 2020-04-05) - judged not relevant; reason summarised from its thread
- P8: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P9: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P10: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P11: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P12: [pgbench - option to build using ppoll() for larger connection counts](https://commitfest.postgresql.org/patch/1307/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P13: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P14: [Remove pgpid_t type, use pid_t instead](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5035c93c8a5ac6804da79c67403460348b381924) (revert, Partially reverted, 2022-10-22) - judged not relevant
- P15: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P16: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P17: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P18: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P19: [Allow WHEN in INSTEAD OF triggers](https://commitfest.postgresql.org/patch/2398/) (cf, Withdrawn, 2020-02-26) - judged not relevant
- P20: [Use simplehash.h instead of dynahash in SMgr](https://commitfest.postgresql.org/patch/3092/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P21: [TAP test module - PostgresClient](https://commitfest.postgresql.org/patch/1442/) (cf, Rejected, 2018-03-03) - judged not relevant
- P22: [Add test module for Table Access Method](https://commitfest.postgresql.org/patch/4588/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P23: [Fix runtime errors from -fsanitize=undefined](https://commitfest.postgresql.org/patch/2144/) (cf, Rejected, 2019-08-13) - judged not relevant
- P24: [Revert "Add missing padding from MemoryChunk struct"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5495796ad12aae971137d184ff25add8f6b03968) (revert, Reverted, 2022-08-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 126 s; cost reported by the CLI: $2.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 75 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:45:28.281Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
