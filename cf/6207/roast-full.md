# Design pre-review: Fix socket and shared memory handle inheritance on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6207](https://commitfest.postgresql.org/patch/6207/) (Needs review); authors: Bryan Green (lispyone)
- Thread: [\[PATCH\] Fix socket handle inheritance on Windows](https://www.postgresql.org/message-id/flat/1387e1ea-ee05-4f5b-a1c4-6c91fddb6d07%40gmail.com), started 2025-11-06
- Messages read: 10 of 10
- Patch prose read: v3-0001-Fix-socket-handle-inheritance-on-Windows.patch, v3-0002-Fix-shared-memory-handle-inheritance-on-Windows.patch, 0001-Fix-socket-handle-inheritance-on-Windows-preventing-.patch.bak (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible failure, Thomas Munro agreed with the analysis in message #3, and v2/v3 follow the SOCK_CLOEXEC and socket_set_cloexec() design he suggested. The design copies existing practice: Unix FD_CLOEXEC in pq_init() and the signal pipe duplication in save_backend_variables(). Confidence is medium because nobody has reviewed v2 or v3 and only 3 of 10 messages are visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody could check whether other WSASocket or CreateProcess callers still create inheritable handles, or what access rights the handle duplicated into each child gets. Messages #4 to #10 were not shown.

## The proposal, restated

**Problem.** On Windows, programs run by a backend (COPY TO PROGRAM, archive_command) inherit the listen sockets, the client socket and the shared memory handle. If the server dies while such a program runs, the port stays bound and the client connection stays open. A new postmaster then fails with "Address already in use" or "pre-existing shared memory block is still in use" until the program exits.

**Proposed design.** 0001 adds a backend-only SOCK_CLOEXEC on Windows, which pgwin32_socket() maps to WSA_FLAG_NO_HANDLE_INHERIT, and uses it for listen sockets. It adds socket_set_cloexec() and uses it in pq_init() on all platforms in place of fcntl(). 0002 keeps the postmaster's shared memory handle non-inheritable and duplicates it into each child in save_backend_variables(), as already done for the signal pipe.

**User-visible surface.**
- C API: SOCK_CLOEXEC macro defined for backend code on Windows, accepted by socket creation via pgwin32_socket().
- C API: new socket_set_cloexec() function next to socket_set_nonblocking() in src/port.
- New TAP test src/test/postmaster/t/005_handle_inheritance.pl.

**What the author claims.**
- The new tests fail on master and pass with the patches.
- Tested with MSVC on Windows 11 across regress, recovery, isolation, postmaster, pg_ctl and libpq suites; v3 passes all CI tasks in the author's fork.
- The shared memory fix covers every child process, not only backends.
- The v1 test used \copy, so the program was psql's child and the test could not catch the bug.
- accept4() support is left out because it only matters for multithreaded backends.

**Thread state.** Thomas Munro agreed with the analysis and suggested the SOCK_CLOEXEC and socket_set_cloexec() design that v2/v3 follow; Harrison Booth tested v1 on Windows x64 and ARM64 CI. Nobody has reviewed v2 or v3 yet, and the question of Harrison's force_initdb supplement is open.

## 2. Prior attempts

### Close backend file handles on exec (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/886/](https://commitfest.postgresql.org/patch/886/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAHyXU0xL7qFCkHt45mfP6VdwoN_j4x8DoX8-_xhSuJ8re7f96g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAHyXU0xL7qFCkHt45mfP6VdwoN_j4x8DoX8-_xhSuJ8re7f96g@mail.gmail.com)
- **Date:** decided 2017-01-31; started 2016-10-13
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped responding to Peter Eisentraut's review, and Michael Paquier marked the entry returned with feedback about three weeks later. The review asked why EXEC_BACKEND was excluded, asked for an FD_CLOEXEC fallback on systems without O_CLOEXEC, a check of other places that open file descriptors, and a test that lists the descriptors an exec'd program inherits. It also noted that a code comment was wrong because archive_command and COPY to program call execve().
- **Reason (quote):** "Oskari, are you planning to answer to this review? As the thread has died 3 weeks ago, I am marking this as returned with feedback." ([message](https://www.postgresql.org/message-id/CAB7nPqTrT79weMTEMBWuqrpBqsevwYKR6gSmO03T4wddL50nPQ%40mail.gmail.com))
- **What is different this time:** P18 marked backend files close-on-exec on Unix and excluded EXEC_BACKEND. This proposal covers Windows sockets and the shared memory handle, and it includes a TAP test that runs a server-side program and shows the leak.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 adds a backend-only SOCK_CLOEXEC macro on Windows and socket_set_cloexec() in src/port; 0002 stops child programs inheriting the shared memory handle. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Programs run by COPY TO PROGRAM and archive_command no longer receive the listen sockets, the client socket or the shared memory handle. |
| backpatchability | **yes** | The entry is tagged Bugfix, but the proposal does not say whether it targets supported branches; 0002 changes the data passed in save_backend_variables(). |
| maintenance burden ("who maintains this in 10 years") | **yes** | New code that creates a socket or a handle on Windows must opt in to SOCK_CLOEXEC or socket_set_cloexec() to keep the fix complete. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the fix meant for back-patching, and if so, to which branches?
2. Message #1 lists pipe handles as affected: will a pipe patch follow in this thread, or is that case no longer a problem?
3. After 0001 and 0002, which handles in a backend on Windows are still inheritable by a program that COPY TO PROGRAM or archive_command starts?
4. Does this patch depend on the file handle patch in CF 6197, or can a committer apply the two in either order?
5. Does replacing fcntl() with socket_set_cloexec() in pq_init() change behaviour in EXEC_BACKEND builds on non-Windows platforms?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Thomas Munro agreed with the analysis and suggested the design in message #3, but nobody has reviewed v2 or v3. The owner needs to know the Windows port code and EXEC_BACKEND child startup.
- **Liability:** The owner signs for postmaster and child startup on Windows: socket creation in pgwin32_socket() and the shared memory handle passed in save_backend_variables(). A mistake there stops every child process from starting. Back-patching would extend that to all supported branches.
- **Author follow-up:** The author posted three versions, reworked the design after message #3, corrected the v1 test that could not catch the bug, and renamed the CommitFest entry on 2026-11-01.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The fix helps Windows servers that run COPY TO PROGRAM or archive_command, and the two patches are small and stay inside Windows port code and child startup. The entry has been in 3 CommitFests and v2/v3 have no review, so it needs a reviewer who can test on Windows.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only related precedent, P18, ended because its author stopped replying to review, and the design was never rejected. Its review points (EXEC_BACKEND, a test of inherited handles) are answered here, and file handles belong to the separate patch in CF 6197. The thread names no other earlier attempt.
- standards-compat: 0 objection(s). The proposal adds no SQL syntax, GUC, catalog object, psql command or dump-visible state. Its one contract change, that programs run by the server no longer inherit Windows socket and shared memory handles, matches what Unix builds already do with FD_CLOEXEC. Not checked: whether any extension depends on inheriting the shared memory handle, and whether back-patching is intended. [P18] concerns file descriptors and stopped for lack of a reply, so it does not apply here.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no on-disk format, WAL, catalog or wire protocol. The struct changed in save_backend_variables() passes only between a postmaster and children of the same binary. The new SOCK_CLOEXEC macro and socket_set_cloexec() add to the API and break no existing extension API, so neither blocks back-patching.
- security: 0 objection(s). The patch narrows what programs started by a backend receive, so it removes exposure and adds none; COPY TO PROGRAM and archive_command already need superuser or pg_execute_server_program. The proposal does not say what access rights the handle duplicated into each child gets, and that can only be checked in the code.
- maintenance-fit: 0 objection(s). No objection from the maintenance angle. The opt-in SOCK_CLOEXEC and socket_set_cloexec() design follows Unix practice and was suggested by Thomas Munro (messages #3 and #5), and 0002 reuses the existing signal-pipe pattern in save_backend_variables(). The changes stay inside Windows port code with no cost on any common path. Two points could not be checked without the code: whether other WSASocket or CreateProcess callers still create inheritable handles, and the pipes listed in message #1, for which no patch is posted. [P18] is a close-on-exec patch for files that was returned after its author stopped responding, not for a design reason.
- already-exists: 0 objection(s). Postgres has no existing feature, setting or extension that stops Windows child programs from inheriting socket and shared memory handles, and an extension cannot change how the postmaster creates them. The design reuses existing mechanisms: the socket_set_nonblocking() pattern, the duplication of the signal pipe in save_backend_variables(), and the SOCK_CLOEXEC idiom that Unix already uses.
- demand: 0 objection(s). This is a bug fix with a reproducible failure. A Windows server killed while COPY TO PROGRAM or archive_command runs cannot restart until that program exits. Thomas Munro accepted the analysis in message #3, and Harrison Booth reproduced the failure on unpatched Windows x64 and ARM64 CI in message #6. The cost falls only on Windows-specific code paths, so the demand angle finds no objection.

### Precedents retrieved

- P1: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P2: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P3: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P4: [Revert "graceful shutdown" changes for Windows.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29992a6a509b256efc4ac560a1586b51a64b2637) (revert, Reverted, 2022-03-22) - judged not relevant
- P5: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P6: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P8: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P10: [Revert "Secure Unix-domain sockets of "make check" temporary clusters."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8f5578d0f9681ef81bc71a3762a191d66a29c8b1) (revert, Reverted, 2014-03-29) - judged not relevant
- P11: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - judged not relevant
- P12: [Revert commits 290acac92b and 8a7e9e9dad.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d) (revert, Reverted, 2019-11-30) - judged not relevant
- P13: [Revert to our pre-7.4 behavior of identifying Unix-socket connections in ps status as '\[local\]', not as 'localhost' as the code has been doing recently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b43e325ebfab423c937ce384bcdaf70ed0b462e) (revert, Reverted, 2003-09-12) - judged not relevant
- P14: [Revert incorrect memory-conservation hack in inheritance_planner().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0f36045b2e3d037bb7647d84373404fa4ba9588) (revert, Reverted, 2011-01-13) - judged not relevant
- P15: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P16: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/1745/) (cf, Returned with feedback, 2019-06-28) - judged not relevant; reason summarised from its thread
- P17: [Remove investigative code for can't-reattach-to-shared-memory errors.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bcbf2346d69f6006f126044864dd9383d50d87b4) (revert, Partially reverted, 2018-05-01) - judged not relevant
- P18: [Close backend file handles on exec](https://commitfest.postgresql.org/patch/886/) (cf, Returned with feedback, 2017-01-31) - used in the review; reason summarised from its thread
- P19: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P20: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/2186/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P21: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant
- P23: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1179/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P24: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 78 s; cost reported by the CLI: $1.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:30.803Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
