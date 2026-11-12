# Design pre-review: Windows relation extension failure at 2GB and 4GB

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6204](https://commitfest.postgresql.org/patch/6204/); [thread](https://www.postgresql.org/message-id/flat/0f238ff4-c442-42f5-adb8-01b762c94ca1%40gmail.com), 32 of 32 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The I/O fix is committed to HEAD and no one disputes the goal. What remains is the test module and the v6 meson.build change, and all three surviving objections are fixable level 2 points. standards-compat-1 is the strongest: v6 removes the meson segment size check while other code still uses off_t. maintenance-fit-2 and maintenance-fit-3 repeat requests from the engaged committer that v6 does not yet meet.

Not checked: The code was not read. What v6 changes in meson.build is known only from the quote in message #28, and how the test behaves on a filesystem without sparse files could not be checked.

## Prior attempts

- [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://www.postgresql.org/message-id/flat/CAKZiRmyM4YnokK6Oenw5JKwAQ3rhP0YTz2T-tiw5dAQjGRXE3Q@mail.gmail.com) (similar idea; Withdrawn, 2024-11-25). The visible messages do not show why. The last message discusses how to back-patch the fix, including an ftruncate() replacement, and nobody objects to it. The entry was withdrawn three days later, and no message gives a reason. (AI summary of the thread) Different now: P1 fixed 32-bit off_t in frontend tar writing (lseek, ftruncate) for pg_basebackup. This proposal fixes the backend fd.c and md.c layer and the OVERLAPPED wrappers. P1 was withdrawn with no recorded reason, so no objection carries over.
- [Large files for relations](https://www.postgresql.org/message-id/flat/CA+hUKG+BGXwMbrvzXAjL8VMGf25y_ga_XnO741g10y0=m6dDiA@mail.gmail.com) (similar idea; Returned with feedback, 2024-10-07). "If I run it with 64kB, the test pgbench/001_pgbench_with_server fails consistently, so it seems there is still a gap somewhere." ([message](https://www.postgresql.org/message-id/9b0235f1-f1f1-47d8-a764-43ea06fdb65f%40eisentraut.org)) Different now: P2 aimed at a per-cluster segment size set at initdb and stalled on incremental backup. This proposal keeps the compile-time segment size and only makes offsets 64-bit on Windows, so P2's recorded problems do not apply.

## Objections a reviewer is likely to raise

### 1. v6 removes the meson segment size check before all relation-file paths are converted

A packager builds with MSVC and -Dsegsize=4, which v6 lets meson accept. The server extends relation files past 2GB, but CREATE DATABASE with file copy, pg_rewind or pg_basebackup handles those files through code that still uses off_t. The failure shows up later, during a copy, rewind or backup.

Message #28 quotes v6 removing the `cc.sizeof('off_t') < 8` check from meson.build. The commit message does not mention this change. The thread names off_t users that remain: buffile.c (message #30), and copydir.c and pg_rewind's libpq_source.c and filemap.c (message #2). configure keeps its own check, so after v6 the two build systems would accept different segment sizes for the same compiler.

**Question:** Which code paths that read, copy or back up relation segment files still use off_t, and can the meson check stay until all of them use pgoff_t?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/aQwsi2IfG0imIwe-%40paquier.xyz), [#28](https://www.postgresql.org/message-id/aS5TIG9BIKrTrZz4%40paquier.xyz), [#30](https://www.postgresql.org/message-id/aS5WzzmLvlXuKbQp%40paquier.xyz); `standards-compat-1`</sub>

### 2. A 4GB sparse-file test runs in every check with no opt-in gate

A buildfarm animal or developer runs make check on a filesystem without sparse file support, or on a small SD card. The test writes past 4GB, so the run fills the disk or becomes very slow. Everyone who runs check pays this cost, but only MSVC builds with a large segment size gain anything.

In message #24, Michael proposed a new PG_TEST_EXTRA value for this reason. The v6 README mentions only NO_INSTALLCHECK and claims the test completes in seconds "on most platforms". Sparse files are a property of the filesystem, and nothing checks for them before the test writes past 4GB. Both the README and message #26 frame the gate as optional.

**Question:** What happens to make check on a filesystem without sparse files, and why should the test not be gated by PG_TEST_EXTRA?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#24](https://www.postgresql.org/message-id/aRVZDTFwNwBUdyGe%40paquier.xyz), [#26](https://www.postgresql.org/message-id/aRbP4uSwd0Edpp5S%40paquier.xyz), [#27](https://www.postgresql.org/message-id/3902ccf2-cf32-4f35-9925-2240e34c9141%40gmail.com); `maintenance-fit-2`</sub>

### 3. Test module is a single fixed scenario, not reusable File* wrappers

The off_t conversion of buffile.c or pg_rewind later needs a test of its own. The hard-coded test_large_files_test_4gb_boundary() cannot express it, so a second module or another fixed C function gets added. The tree then holds several narrow I/O test modules for one problem.

Michael asked in message #9 and again in message #28 for thin SQL wrappers around File* functions that take bytea and an offset. The v6 test is one C function that runs a fixed sequence. It also checks sizeof(pgoff_t), which is 8 bytes in every build (message #28), so that check can never skip anything. No later version shows the requested design.

**Question:** Can the module expose open, write, read and size wrappers that take an offset and bytea, so that later off_t conversions reuse it?

<sub>fixable, level 2, ABI / extension API; messages [#9](https://www.postgresql.org/message-id/aQ0djavBB5G4QMbS%40paquier.xyz), [#28](https://www.postgresql.org/message-id/aS5TIG9BIKrTrZz4%40paquier.xyz); `maintenance-fit-3`</sub>

## Questions for the author

1. Does v6 remove the meson check that rejects a segment size of 2GB or more when off_t is under 8 bytes, and if so, why?
2. Which code paths that read, copy or back up relation segment files still use off_t, and when will they move to pgoff_t?
3. What happens to make check on a filesystem without sparse files, and why is the test not gated by a PG_TEST_EXTRA value?
4. Can the module expose open, write, read and size wrappers that take an offset and bytea, as message #28 asks?
5. Why does the v6 commit message describe the I/O fix when its file list holds only meson.build and the test module?

## Touch points

- **ABI / extension API:** File* functions in fd.h and pg_pread(), pg_pwrite(), pg_preadv(), pg_pwritev() change their offset type from off_t to pgoff_t.
- **backpatchability:** The signature changes went to HEAD only because of ABI risk; only the meson segment size check was backpatched to v16.
- **maintenance burden ("who maintains this in 10 years"):** A new test module, test_large_files, and an off_t conversion that is only partly done across the tree.
- **performance on the hot path:** The patch changes offset types in fd.c, md.c and the pread/pwrite wrappers, which every relation read and write uses.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier committed the meson check and the I/O changes and reviews the test module in message #28. The owner needs to know the Windows port and the fd.c layer.
- Priority: proceed; audience niche; review cost low. The fix helps only Windows MSVC builds with a segment size above 1GB, but it is already committed and the rest is a small test module. A committer is engaged and has listed concrete changes, so a v7 that answers message #28 can move forward.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
