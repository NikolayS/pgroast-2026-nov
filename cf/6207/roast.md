# Design pre-review: Fix socket and shared memory handle inheritance on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6207](https://commitfest.postgresql.org/patch/6207/); [thread](https://www.postgresql.org/message-id/flat/1387e1ea-ee05-4f5b-a1c4-6c91fddb6d07%40gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible failure, Thomas Munro agreed with the analysis in message #3, and v2/v3 follow the SOCK_CLOEXEC and socket_set_cloexec() design he suggested. The design copies existing practice: Unix FD_CLOEXEC in pq_init() and the signal pipe duplication in save_backend_variables(). Confidence is medium because nobody has reviewed v2 or v3 and only 3 of 10 messages are visible.

Not checked: Without the code, nobody could check whether other WSASocket or CreateProcess callers still create inheritable handles, or what access rights the handle duplicated into each child gets. Messages #4 to #10 were not shown.

## Prior attempts

- [Close backend file handles on exec](https://www.postgresql.org/message-id/flat/CAHyXU0xL7qFCkHt45mfP6VdwoN_j4x8DoX8-_xhSuJ8re7f96g@mail.gmail.com) (similar idea; Returned with feedback, 2017-01-31). "Oskari, are you planning to answer to this review? As the thread has died 3 weeks ago, I am marking this as returned with feedback." ([message](https://www.postgresql.org/message-id/CAB7nPqTrT79weMTEMBWuqrpBqsevwYKR6gSmO03T4wddL50nPQ%40mail.gmail.com)) Different now: P18 marked backend files close-on-exec on Unix and excluded EXEC_BACKEND. This proposal covers Windows sockets and the shared memory handle, and it includes a TAP test that runs a server-side program and shows the leak.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is the fix meant for back-patching, and if so, to which branches?
2. Message #1 lists pipe handles as affected: will a pipe patch follow in this thread, or is that case no longer a problem?
3. After 0001 and 0002, which handles in a backend on Windows are still inheritable by a program that COPY TO PROGRAM or archive_command starts?
4. Does this patch depend on the file handle patch in CF 6197, or can a committer apply the two in either order?
5. Does replacing fcntl() with socket_set_cloexec() in pq_init() change behaviour in EXEC_BACKEND builds on non-Windows platforms?

## Touch points

- **ABI / extension API:** 0001 adds a backend-only SOCK_CLOEXEC macro on Windows and socket_set_cloexec() in src/port; 0002 stops child programs inheriting the shared memory handle.
- **security / privileges:** Programs run by COPY TO PROGRAM and archive_command no longer receive the listen sockets, the client socket or the shared memory handle.
- **backpatchability:** The entry is tagged Bugfix, but the proposal does not say whether it targets supported branches; 0002 changes the data passed in save_backend_variables().
- **maintenance burden ("who maintains this in 10 years"):** New code that creates a socket or a handle on Windows must opt in to SOCK_CLOEXEC or socket_set_cloexec() to keep the fix complete.

## Who would commit it, and when

- Owner: qualified committer engaged. Thomas Munro agreed with the analysis and suggested the design in message #3, but nobody has reviewed v2 or v3. The owner needs to know the Windows port code and EXEC_BACKEND child startup.
- Priority: needs champion; audience niche; review cost medium. The fix helps Windows servers that run COPY TO PROGRAM or archive_command, and the two patches are small and stay inside Windows port code and child startup. The entry has been in 3 CommitFests and v2/v3 have no review, so it needs a reviewer who can test on Windows.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
