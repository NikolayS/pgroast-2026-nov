# Design pre-review: Fix a file descriptor leak when using io_method=io_uring

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6617](https://commitfest.postgresql.org/patch/6617/); [thread](https://www.postgresql.org/message-id/flat/zHN8Rb7syWYDBPUhCICxKLr_tQmFLg0T024irxVVb6RSSicZcqcA0eT73z2PQy7JmzzqnUzX0Hi3P2dPKRvmNx_jbAqSCLyqJvWjXEypyAI%3D%40draescher.fr), 22 of 22 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproduction, several reviewers confirmed the leak, and the fix follows the existing pgaio_worker_die() pattern in method_worker.c. A committer reports the v1-based fix pushed to master and 18, which leaves IoMethodOps unchanged. Confidence is medium because only 3 of 22 messages and no code were read.

Not checked: The code and messages #4 to #22 were not read, so the callback's behaviour in the postmaster and the pushed commit could not be checked. The follow-up that depends on liburing internals was not judged.

## Prior attempts

None found before 2026-03-18.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which commit and which branches carry the fix, so the CommitFest entry can be closed as committed?
2. Does liburing allow io_uring_queue_exit() on a postmaster-local copy of struct io_uring, or does the follow-up need another way to keep the ring descriptors?
3. Can a test kill a backend with io_method=io_uring and check that the postmaster's io_uring fd count stays the same after the crash restart?
4. Will the shmem_cleanup member of IoMethodOps still be proposed for master, and which second IO method would use it?

## Touch points

- **ABI / extension API:** v2 to v4 add a shmem_cleanup member to IoMethodOps in aio_internal.h; the committed v1-based fix does not.
- **backpatchability:** The fix targets REL_18_STABLE; v3 and v4 would break the Postgres 18 ABI, so the v1 shape was used.
- **maintenance burden ("who maintains this in 10 years"):** The postmaster now calls io_uring_queue_exit() at crash restart, and the planned follow-up depends on liburing internals.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao is listed as committer on the CommitFest entry and reports the v1-based fix pushed. The follow-up needs someone who knows the AIO io_uring code and postmaster crash restart.
- Priority: proceed; audience a sizeable minority; review cost low. The fix is a small change in one file for a confirmed leak that can stop a crash restart with io_method=io_uring, and it is reported pushed. What remains is to close the entry and review the follow-up for postmaster-local ring descriptors as a separate patch.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
