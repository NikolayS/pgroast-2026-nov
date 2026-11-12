# Design pre-review: Fix pg_restore gzip stream validation (bug #19754)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7456](https://commitfest.postgresql.org/patch/7456/) (Needs review); authors: Chao Li (chaol)
- Thread: [Fix pg_restore gzip stream validation (bug #19754)](https://www.postgresql.org/message-id/flat/C482E34B-347F-4AD2-ACE1-96142F6088B2%40gmail.com), started 2026-10-14
- Messages read: 5 of 5
- Patch prose read: v2-0001-Add-a-TAP-test-for-pg_restore-bug-19754.patch, v2-0002-Fix-gzip-stream-validation-in-pg_restore.patch, v1-0001-Add-a-TAP-test-for-pg_restore-bug-19754.patch, v1-0002-Fix-gzip-stream-validation-in-pg_restore.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and the reporter confirmed the fix on two platforms with three zlib versions, so nothing fails at level 1. Two fixable design points survive, and both concern the back-patch that message #3 asks for. maintenance-fit-1: v2-0002 bundles the pg_restore fix with a refactor of astreamer_gzip, which other tools use. standards-compat-1: zero-byte files and trailing bytes become errors. A committer who knows the frontend compression code has to decide the back-branch shape.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check whether the refactored astreamer_gzip behaves the same for the other tools, or whether memory use stays bounded on concatenated or malformed gzip members.

## The proposal, restated

**Problem.** With zlib 1.3.2, gzread() and gzclose() can return wrong results, so pg_restore rejects valid gzip dump files whose sizes end on buffer boundaries and can accept truncated ones. This hits users restoring gzip-compressed dumps, including the Windows binaries of 14 to 18, which ship zlib 1.3.2.

**Proposed design.** The patch moves the gzip decompression logic out of astreamer_gzip into a shared frontend helper in src/fe_utils/gzip.c that calls zlib's inflate() and lets callers handle errors. astreamer_gzip and pg_restore's compress_gzip.c both use it. pg_restore validates the stream through EOF, checks unread data at close, and supports concatenated gzip members.

**User-visible surface.**
- pg_restore now rejects zero-byte gzip data files and trailing bytes after the last gzip member, per the tester's reading of the stricter parsing.
- pg_restore reports a new truncation error, "compressed stream is incomplete".
- New frontend C header src/include/fe_utils/gzip.h with a shared gzip decompression helper.

**What the author claims.**
- The new TAP test 007_pg_dump_gzip.pl fails with zlib 1.3.2 without the fix and passes with it.
- astreamer_gzip's push interface does not fit pg_restore's read interface, so the patch extracts a shared helper instead of reusing astreamer_gzip.
- The reporter confirmed the fix on Debian 13 and Windows 11 with zlib 1.3.1, 1.3.2 and develop, including parallel restore, empty tables and large objects.
- The reporter says pg_dump always writes at least an empty member, so the stricter parsing does not break dumps it produced.
- The reporter asks for a back-patch to 14, noting that 14 and 15 lack compress_gzip.c and need their own fix.

**Thread state.** The reporter tested v2 and asked for a back-patch. Daniel Gustafsson said he has the patch on his TODO list. No committer review or back-patch decision is visible.

## 2. Prior attempts

### Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4)
- **Thread:** [https://www.postgresql.org/message-id/flat/20231231.200741.1078989336605759878.horikyota.ntt@gmail.com](https://www.postgresql.org/message-id/flat/20231231.200741.1078989336605759878.horikyota.ntt@gmail.com), [https://www.postgresql.org/message-id/flat/20220925232237.p6uskba2dw6fnwj2@awork3.anarazel.de](https://www.postgresql.org/message-id/flat/20220925232237.p6uskba2dw6fnwj2@awork3.anarazel.de)
- **Date:** decided 2024-01-23; started 2023-01-24
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 728f86fec65537eade8d9e751961782ddb527934. The signal handling was a few bricks shy of a load in that commit, which made the walreceiver non-responsive to SIGTERM while it was waiting for the connection to be established. That prevented a standby from being promoted. Since it was non-essential refactoring, let's revert it to make v16 work the same as earlier releases. I reverted it in 'master' too, to keep the branches in sync. The refactoring was a good idea as such, but it needs a bit more work. Once we have developed a complete patch with this issue fixed, let's re-apply that to 'master'.
- **Reason (quote):** "Since it was non-essential refactoring, let's revert it to make v16 work the same as earlier releases."
- **What is different this time:** P18 was a pure refactor of libpqwalreceiver. This proposal is a bug fix that bundles a refactor of shared frontend code used by other tools.


## 3. Objections a reviewer is likely to raise

### 1. Back-patched bugfix also refactors astreamer_gzip, used by other tools

fixable · level 2, rubric item 9 · backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A packager ships a 16-18 minor release with this patch to fix pg_restore. The astreamer_gzip rework changes how pg_basebackup, pg_verifybackup or pg_waldump handle gzip input. A user finds this only when a backup check or WAL read starts to fail after a minor update.

Message #3 asks for a back-patch to 14, and v2-0002 moves the inflate logic out of astreamer_gzip into a new src/fe_utils/gzip.c. That moves code used by tools that have no reported bug. [P18] reverted a non-essential refactor from a released branch for this reason. 14 and 15 lack compress_gzip.c (message #3), so master's patch shape does not carry over, and the project would keep two or three versions of the fix.

**Precedents.**
- [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (Reverted, 2024-01-23); thread: [#1](https://www.postgresql.org/message-id/flat/20231231.200741.1078989336605759878.horikyota.ntt@gmail.com), [#2](https://www.postgresql.org/message-id/flat/20220925232237.p6uskba2dw6fnwj2@awork3.anarazel.de)

Thread: [#1 (Chao Li, 2026-10-14)](https://www.postgresql.org/message-id/C482E34B-347F-4AD2-ACE1-96142F6088B2%40gmail.com), [#3 (, 2026-10-16)](https://www.postgresql.org/message-id/001b01dd5d3f%24c6a15e90%2453e41bb0%24%40gmx.net)

**Question:** Can the series be split into a minimal back-patchable pg_restore fix and a separate master-only astreamer_gzip refactor, and what does the fix look like for 14 and 15?

Counter-check: held. The thread asks for a back-patch but gives no plan, and 0002 changes astreamer_gzip, which only recent branches have in fe_utils. [P18] concerns a different subsystem and gives only general support, but the objection holds without it.

### 2. Stricter gzip parsing rejects files that pg_restore accepts today

fixable · level 2, rubric item 6 · backpatchability, pg_dump / restore round-trip · `standards-compat-1`

A user keeps a directory-format dump that contains a zero-byte data file, or whose gzip files carry trailing padding from a copy or archive tool. After a minor upgrade that includes this fix, pg_restore stops with an error. The user finds out during a restore, possibly one run to recover lost data.

Message #3 confirms that zero-byte files and trailing bytes after the last member are now errors. The only argument offered is that current pg_dump always writes at least an empty member. That covers newly written dumps, but not dumps from older pg_dump releases or files that have been handled after pg_dump wrote them. gzip(1) accepts trailing zeros with a warning. A fix meant for back branches would likely be asked to keep the old acceptance or downgrade these cases to warnings.

Thread: [#3 (, 2026-10-16)](https://www.postgresql.org/message-id/001b01dd5d3f%24c6a15e90%2453e41bb0%24%40gmx.net)

**Question:** Has every pg_dump release that can write gzip directory or custom archives always written a non-empty gzip member, and why should trailing bytes be an error and not a warning?

Counter-check: held. Message #3 confirms that zero-byte files and trailing bytes become errors, and the thread only covers dumps written by current pg_dump. zlib's gzread() ignores extra trailing data today, so this changes behaviour in a minor release.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a shared frontend helper in src/include/fe_utils/gzip.h and changes astreamer_gzip to use it. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_restore now rejects zero-byte gzip data files and trailing bytes after the last gzip member. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #3 asks for a back-patch to 14; 14 and 15 lack compress_gzip.c, and nls.mk changes touch four tools. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Postgres takes over gzip stream handling from zlib's gzread(), and back branches may need two or three versions of the fix. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the series be split into a minimal pg_restore-only fix for back branches and a master-only refactor of astreamer_gzip?
2. What does the fix look like for 14 and 15, which lack compress_gzip.c?
3. Has every pg_dump release that writes gzip archives always written at least one gzip member, so that no existing dump contains a zero-byte data file?
4. Why are trailing bytes after the last gzip member an error and not a warning, given that pg_restore accepts them today?
5. Does the refactored astreamer_gzip change behaviour or add translatable messages in pg_basebackup, pg_verifybackup or pg_waldump?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Daniel Gustafsson said the patch is on his TODO list, but no committer review is visible yet. The owner needs to know pg_dump's compression code and the astreamer code.
- **Liability:** The owner signs for gzip reading in pg_restore on every branch that gets the fix, and for astreamer_gzip in pg_basebackup, pg_verifybackup and pg_waldump if the refactor is included. Dump files live for years, so a wrongly rejected archive can surface long after the release.
- **Author follow-up:** The author posted v2 one day after v1 to fix CI failures and asked the reporter to test.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Message #3 reports a production failure, and a truncated restore that exited 0, with zlib 1.3.2, which the Windows binaries of 14 to 18 ship. The 35 KiB fix touches pg_dump and shared fe_utils code, and back branches need their own versions.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Bug fix bundles a refactor of astreamer_gzip shared by other tools. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about the astreamer_gzip refactor in back branches and adds the request to split the series.
- `standards-compat-2` Back-patched bug fix also changes astreamer_gzip used by other tools. Refuted (not specific): Duplicate of maintenance-fit-1. Its point about new untranslated messages in back branches is valid and belongs with that objection.
- `replication-ondisk-1` Stricter gzip parsing may reject dump files that already exist. Refuted (not specific): Duplicate of standards-compat-1. Its external recompression example is also weaker, because a gzip tool writes valid members without trailing bytes.
- `replication-ondisk-2` Back-patch also changes the gzip path for base backups. Refuted (not specific): Duplicate of maintenance-fit-1. Its scenario of a subtle bug in the moved code is speculative, and the request for a minimal back-branch fix is the same.

### Review angles

- prior-attempts: 1 objection(s). No precedent rejects the idea of replacing gzread()/gzclose() with inflate() in pg_restore. P3's zlib-variant risk applies only to test reproducibility, which is out of scope here.
- standards-compat: 2 objection(s). The bug itself is real and shown by the reporter's tests. These objections concern how far the behaviour change reaches, mainly in back branches.
- replication-ondisk: 2 objection(s). The patch adds no WAL, catalog, wire-protocol or server-side on-disk change. The objections above concern only existing dump and backup files read by frontend tools.
- security: 0 objection(s). The patch changes how the client-side pg_restore checks gzip streams. It adds no privilege, role, server code or exposed data, and it now rejects truncated dumps that were silently accepted, so data integrity improves. Without the code, memory limits on concatenated or malformed members could not be checked.
- maintenance-fit: 1 objection(s). The bug is real and confirmed by the reporter on two platforms, and a committer has said he will look at it. The only maintenance concern is the patch shape for back branches.
- already-exists: 0 objection(s). No existing feature covers this. The patch fixes wrong results from gzread() and gzclose() in zlib 1.3.2. Pinning or reverting zlib in the Windows packages, or running gunzip on directory-format files by hand, are workarounds for packagers and users, not a Postgres alternative. The patch also reuses the existing astreamer_gzip inflate() logic through a shared helper, so it does not add a second mechanism.
- demand: 0 objection(s). Demand is real. This is a bug fix for a field report: message #3 describes the failure seen in production with shipped Windows binaries of 14 to 18, and a restore that loaded 10073 of 20001 rows and exited 0. Users of gzip dumps with zlib 1.3.2 are a broad group, and no feature surface is added for a small group to pay for.

### Precedents retrieved

- P1: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P2: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P3: [Revert "Fix corner case with PGP decompression in pgcrypto"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=38f60f174e3279069b5547b5f4015eb4a8492037) (revert, Reverted, 2020-07-22) - used in the review
- P4: [Improve pg_restore toc file parsing and format for better performances](https://commitfest.postgresql.org/patch/4509/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P5: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P6: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P7: [Fix pg_restore --no-globals SQL output for pg_dumpall archives](https://commitfest.postgresql.org/patch/6899/) (cf, Withdrawn, 2026-06-20) - judged not relevant; reason summarised from its thread
- P8: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P9: [pg_restore cleanups](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=40b9c27014d90d77c61e8f5c77ddeb84fa9a6b69) (revert, Partially reverted, 2025-04-16) - judged not relevant
- P10: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P11: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P12: [pg_dump: fix NOT NULL constraint name comparison using makeObjectName](https://commitfest.postgresql.org/patch/6601/) (cf, Withdrawn, 2026-10-04) - judged not relevant
- P13: [Detect supported SET parameters when pg_restore is run](https://commitfest.postgresql.org/patch/803/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P14: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P15: [Refactor code around GUC default_toast_compression](https://commitfest.postgresql.org/patch/6727/) (cf, Withdrawn, 2026-09-16) - judged not relevant
- P16: [Fix minor issues in astreamer_zstd.c](https://commitfest.postgresql.org/patch/6386/) (cf, Withdrawn, 2026-01-10) - judged not relevant
- P17: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P18: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - used in the review
- P19: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P20: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant
- P21: [Compression dictionaries](https://commitfest.postgresql.org/patch/3626/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P22: [Revert cascading of JSON_TABLE's ON ERROR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=af6fad879fbf9b542cfde05c52695c9dffa6bb47) (revert, Reverted, 2026-07-14) - judged not relevant
- P23: [Revert "Change default value of default_toast_compression to "lz4""](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f0b3afab4da87255ee159b96ba531157ee5af3b) (revert, Reverted, 2026-03-04) - judged not relevant
- P24: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 109 s; cost reported by the CLI: $1.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 77 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:56:50.235Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
