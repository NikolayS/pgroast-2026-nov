# Design pre-review: aio: Async fsyncs for crash recovery and checkpointer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7190](https://commitfest.postgresql.org/patch/7190/) (Needs review); authors: Nazir Bilal Yavuz (nbyavuz)
- Thread: [aio: Async fsyncs for crash recovery and checkpointer](https://www.postgresql.org/message-id/flat/CAN55FZ0vLWJQNB%3DHuHXG2wabFjXJd6OWTa3%3DkRzwObdZD9poHQ%40mail.gmail.com), started 2026-08-25
- Messages read: 9 of 9
- Patch prose read: v3-0001-ci-Temporarily-enable-fsync-everywhere.patch, v3-0002-aio-Add-fsync-support.patch, v3-0003-aio-worker-Bound-SMGR-cache-growth.patch, v3-0004-aio-Issue-SyncDataDirectory-fsyncs-through-AIO.patch, v3-0005-aio-Allow-AIO-reopen-callbacks-to-return-errors.patch, v3-0006-aio-Issue-checkpointer-fsyncs-asynchronously.patch, v3-0007-aio-Allow-IO-workers-to-execute-SLRU-fsyncs.patch, v3-0008-aio-doc-Document-AIO-fsync.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal of overlapping checkpoint fsyncs, and all four are fixable. The largest risk is replication-ondisk-1: the new concurrent error path must still PANIC on any fsync failure, and no test injects one. already-exists-2 and maintenance-fit-2 ask for measurements at default checkpoint_flush_after and of foreground latency. already-exists-1 affects only the SyncDataDirectory() patch. Only 3 of 9 messages were visible.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the PANIC and retry rules in the 0006 commit message, completion ordering and restartpoint behaviour on standbys are unchecked. Messages #4 to #9 were seen only through the objections that cite them.

## The proposal, restated

**Problem.** The checkpointer's ProcessSyncRequests() and the startup process's SyncDataDirectory() fsync files one at a time, so checkpoints and crash-recovery startup wait on each sync in turn. Storage could run these syncs concurrently. This matters most for clusters with many dirty files at checkpoint.

**Proposed design.** The patch adds PGAIO_OP_FSYNC to the AIO subsystem and converts SyncDataDirectory(), checkpointer sync requests and SLRU syncs to keep a bounded set of fsyncs in flight. Relation and SLRU files can be reopened by I/O workers through the smgr target and a new FileTag target. Data-directory files are path-only, so the worker method runs them synchronously and only io_uring overlaps them.

**User-visible surface.**
- pg_aios: new operation value fsync, with off and length NULL for it.
- pg_aios: new target values sync and sync_filetag.
- New wait event DATA_DIR_SYNC; SLRU_FLUSH_SYNC is now reported by the I/O worker, not the checkpointer.
- pg_stat_io fsync_time now counts submission plus wait time for async fsyncs.
- Sync handler API (SyncOps) reshaped so handlers open the file, assign an AIO target and start the fsync.

**What the author claims.**
- Checkpoint with 4000 dirty relation files: about 2.7x to 3.5x faster with worker, about 3.9x with io_uring.
- SyncDataDirectory() with 4000 relations: about 4.2x to 4.5x faster with io_uring, little gain with worker.
- SLRU checkpoint: about 1.4x to 2.6x faster.
- Author agrees recovery_init_sync_method=syncfs is better than or on par with AIO fsync for crash recovery, so the checkpoint is the main benefit.
- Concurrent fsyncs competing with reads in the worker pool can be helped by adding more I/O workers.

**Thread state.** Nitin Jadhav and Yuhang Qiu reviewed; v3 addresses fsync_time accounting, reopen errno handling, the descriptor cap and the fsync=off to on reload race. Open items are missing tests, including worker-side SLRU coverage, and Yuhang's concern about storage bandwidth contention, on which the author asked for detail.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Checkpoint gain is measured with checkpoint_flush_after disabled

fixable · level 1b, rubric item 2 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-2`

A Linux user runs the default checkpoint_flush_after, so writeback starts during the write phase and the final fsyncs find little dirty data. They upgrade and see a much smaller checkpoint speedup than the 2.7x to 3.9x reported. Meanwhile they take on new behaviour in sync.c and its error paths.

checkpoint_flush_after already exists to make the checkpoint's final fsyncs cheap. It does this by asking the kernel to start writeback early. Every checkpoint benchmark in message #1 sets checkpoint_flush_after = 0, which turns off that feature. So the numbers compare the patch against a configuration that most Linux installations do not use, and they do not show what the patch adds beyond the existing setting.

Thread: [#1 (Nazir Bilal Yavuz, 2026-08-25)](https://www.postgresql.org/message-id/CAN55FZ0vLWJQNB%3DHuHXG2wabFjXJd6OWTa3%3DkRzwObdZD9poHQ%40mail.gmail.com)

**Question:** What checkpoint speedup does the patch give over the default checkpoint_flush_after setting on Linux, for the same 4000-file workload?

Counter-check: held. All checkpoint benchmarks in message #1 set checkpoint_flush_after = 0, while the Linux default starts writeback during the write phase. The thread has no result at the default setting, so the gain most Linux users would see is not shown.

### 2. Async SyncDataDirectory() duplicates recovery_init_sync_method=syncfs on Linux

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A Linux user with slow crash-recovery startup sets io_method=io_uring and gets about the speed that recovery_init_sync_method=syncfs already gives. The project now keeps a new AIO path in fd.c, a PGAIO_TID_SYNC target, a new wait event and a doc paragraph, all for one case where an existing setting matches it.

io_uring is Linux-only, and syncfs is available on Linux. In message #5 the author measured syncfs as better than or equal to the AIO path, and in messages #6 and #8 both sides agreed the main benefit is the checkpoint. Under the worker method, patch 0004 runs these fsyncs synchronously anyway. The case it still helps is a data directory on a filesystem shared with other busy data, where syncfs also flushes unrelated files, and the thread has no measurement of that case.

Thread: [#5 (Nazir Bilal Yavuz, 2026-09-18)](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#6 (Yuhang Qiu, 2026-09-22)](https://www.postgresql.org/message-id/C24C12AB-DE89-4EC8-AF08-6157CDA9A8CA%40gmail.com), [#8 (Nazir Bilal Yavuz, 2026-09-30)](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com)

**Question:** On which Linux setups is syncfs unsuitable but io_uring AIO fsync faster, and could patch 0004 be dropped from the series without hurting the checkpoint work?

Counter-check: held. Message #5 shows syncfs equal to or faster than io_uring fsync, and the author agrees in message #8 that the checkpoint is the main benefit. Since fsync is the default method, some users gain, but the thread does not show how many.

### 3. Concurrent fsync error paths change PANIC handling but are untested

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

During a checkpoint, one in-flight fsync fails with a non-PANIC error and the cleanup drains the others, one of which returns EIO. If that EIO is classed as a retryable failure, the next checkpoint's fsync succeeds on clean pages. The redo pointer then moves past WAL that the lost writes still need, and nothing reports it.

The v3-0006 commit message replaces the serial loop with new rules: preserve the first error that requires PANIC, leave other errors to the next checkpoint, and skip reinterpreting handled deletions during cleanup. A wrong outcome here repeats the loss case that made Postgres PANIC on fsync failure. The worker path adds a reopen-and-fsync in another process plus -errno passing (v3-0005), so the error reaches the checkpointer by a new route. Message #9 and the brief say no tests exist yet, and the thread runs no failing fsync, so the invariant is argued but not shown.

Thread: [#5 (Nazir Bilal Yavuz, 2026-09-18)](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#8 (Nazir Bilal Yavuz, 2026-09-30)](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com), [#9 (Nazir Bilal Yavuz, 2026-09-30)](https://www.postgresql.org/message-id/CAN55FZ2i-%3D87NWg8v66hbbm18o%2BkBmCjyLyX4fE7s49c9JbEBw%40mail.gmail.com)

**Question:** Which fault-injection tests show that EIO from any in-flight fsync, under worker and io_uring, during a checkpoint or restartpoint, still leads to PANIC?

Counter-check: held. The 0006 commit message states the PANIC rule, but message #5 says the -errno route loses error information and is unfinished. No failing fsync has been run under worker or io_uring, so the invariant is stated and not yet shown.

### 4. Checkpoint fsync bursts share the default worker pool with reads

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An OLTP cluster runs the default io_method=worker. At each checkpoint's sync phase, up to io_max_concurrency fsyncs fill the worker queue and the device, and foreground reads queue behind them. Users see read latency spikes at every checkpoint and have no setting that caps fsync concurrency separately.

Yuhang raised storage bandwidth contention in message #4 and again in message #6. The author's answer in message #5 is to add more I/O workers, and message #6 replies that this does not address device bandwidth. The benchmarks run on an otherwise idle system with checkpoint_flush_after = 0. That setting moves writeback into the fsyncs and makes serial fsync look worst, so the cost to non-users and the gain under default flush settings both go unmeasured.

Thread: [#4 (Yuhang Qiu, 2026-09-03)](https://www.postgresql.org/message-id/D707392D-96FF-4867-943D-564DAD6644B3%40gmail.com), [#5 (Nazir Bilal Yavuz, 2026-09-18)](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#6 (Yuhang Qiu, 2026-09-22)](https://www.postgresql.org/message-id/C24C12AB-DE89-4EC8-AF08-6157CDA9A8CA%40gmail.com), [#8 (Nazir Bilal Yavuz, 2026-09-30)](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com)

**Question:** What happens to foreground read latency during a checkpoint with default checkpoint_flush_after and io_method=worker, and how can an operator limit fsync concurrency?

Counter-check: held. Yuhang's bandwidth point in message #6 is unanswered: the author asked for detail in message #8, and no benchmark measures foreground latency. io_max_concurrency is the only cap, and it also limits reads.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: Concurrent fsync error paths change PANIC handling but are untested |
| WAL / replication / standby | **yes** | Checkpoints and restartpoints advance the redo pointer only after these fsyncs succeed, and the patch rewrites that error path. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch reshapes SyncOps, adds PGAIO_OP_FSYNC and two AIO targets, and lets reopen callbacks return -errno. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | sync.c gains in-flight bookkeeping, deferred completion and first-PANIC-wins rules that later changes must keep correct. |
| performance on the hot path | **yes** | Under the default io_method=worker, checkpoint fsyncs share the I/O worker pool and the device with foreground reads. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The SyncDataDirectory() conversion overlaps with recovery_init_sync_method=syncfs, which the author measured as equal or faster. |

## 5. Questions the author must answer before anyone reviews code

1. What checkpoint speedup does the patch give with the default checkpoint_flush_after on Linux, for the same 4000-file workload?
2. Which fault-injection tests show that EIO from any in-flight fsync, under worker and io_uring, in a checkpoint or restartpoint, still leads to PANIC?
3. What happens to foreground read latency during a checkpoint with io_method=worker and default flush settings, on slower or shared storage?
4. How can an operator limit checkpoint fsync concurrency without also lowering io_max_concurrency for reads?
5. On which Linux setups is syncfs unsuitable but io_uring fsync faster, and can the SyncDataDirectory() patch be dropped without hurting the checkpoint work?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows both the AIO subsystem and the checkpointer's sync.c fsync error handling. The visible thread and the CommitFest entry show no committer taking part.
- **Liability:** The owner signs for fsync failure handling in every checkpoint and restartpoint, under all three io_method values, with no time limit. A lost fsync error shows only after a later crash.
- **Author follow-up:** The author answered each review point in message #3 and posted v3 with fixes in message #8, so he is active during review. The thread has no sign either way about post-commit fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Clusters with many dirty files per checkpoint gain, but the size of the gain at default flush settings is not shown. The series is about 125 KiB across AIO, sync.c, smgr, SLRU and fd.c, and it changes durability error paths, so review and upkeep are costly. The thread is active and the open points are fixable.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Concurrent checkpoint fsyncs become default with no opt-out short of io_method=sync. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same storage contention concern and also covers the shared worker pool and the missing latency measurement.
- `standards-compat-2` Checkpointer fsync wait events move to I/O workers or generic AIO events. Refuted (no failure scenario): AIO reads already report their wait events from the I/O worker, or as generic AIO events under io_uring, so the patch follows existing practice. The missing documentation is a level 4 matter.
- `maintenance-fit-1` Rewritten fsync error and PANIC path has no committer engaged. Refuted (not specific): Duplicate of replication-ondisk-1 for the untested error path. The lack of an engaged committer belongs in the committability field and does not count against the design.
- `maintenance-fit-3` Crash-recovery conversion adds a target and docs for little gain. Refuted (not specific): Duplicate of already-exists-1. Its claim that the path-only target exists mainly for startup is also weak, because SLRUs not registered with sync.c use PGAIO_TID_SYNC too.
- `demand-1` Data-directory fsync path helps only where syncfs already does as well. Refuted (not specific): Duplicate of already-exists-1, which makes the same syncfs comparison and also names the case where syncfs is unsuitable.
- `demand-2` Checkpoint gain shown only with checkpoint_flush_after = 0. Refuted (not specific): Duplicate of already-exists-2, which asks for the same default-setting measurement more concretely.

### Review angles

- prior-attempts: 0 objection(s). No precedent found with the same idea. [P6] built the AIO subsystem and lapsed from inactivity, [P1] was withdrawn as redundant, and [P4] concerned WAL rows in pg_stat_io; none of these reasons applies here. Message #1 says the idea comes from Andres Freund's earlier branch work, and the thread reports no rejection of it.
- standards-compat: 2 objection(s). The pg_stat_io fsync_time semantics were settled in messages #2 to #7 by following the async-read accounting. The NULL off and length values in pg_aios follow the existing view. The SyncOps reshape is internal because sync handlers are a fixed table.
- replication-ondisk: 1 objection(s). The proposal changes no on-disk format, WAL record, catalog or wire protocol, and the SyncOps and AIO target changes are internal to a major release. The fsync=off to on reload race was settled in v3 (messages #7 and #8). Checkpoint completion ordering and standby restartpoints share the same path and cannot be checked without the code.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The new fsyncs run only in the checkpointer, the startup process and I/O workers, on files these processes already sync, and users gain no new command, function or default. The one new user-visible data is the SLRU name and segment number shown in pg_aios, which is not sensitive. The fsync=off to on reload race was a durability issue, and v3 fixed it by capturing the issuer's setting. Foreground latency during fsync bursts is a performance question for another angle.
- maintenance-fit: 3 objection(s). Objections listed; the code was not reviewed, so struct size and per-handle cost of the FileTag target could not be checked.
- already-exists: 2 objection(s). No extension or contrib module overlaps the checkpoint part. The objections cover the SyncDataDirectory() overlap with syncfs and the existing checkpoint_flush_after mechanism.
- demand: 2 objection(s). The checkpoint part addresses a known serial bottleneck, and two reviewers support the direction. The objections ask for evidence about the startup path and about default settings. They do not reject the idea.

### Precedents retrieved

- P1: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - used in the review; reason summarised from its thread
- P2: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P3: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - used in the review; reason summarised from its thread
- P5: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P7: [Revert initdb --sync-only patch that had incorrect commit messages.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=25d1ed04a2c4d176f6428b9eed57344e50f44d77) (revert, Reverted, 2012-12-04) - judged not relevant
- P8: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P9: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant; reason summarised from its thread
- P10: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P11: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P12: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P13: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant
- P14: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant
- P15: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P16: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant
- P17: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P18: [Asynchronous execution of ForeignScans](https://commitfest.postgresql.org/patch/751/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P19: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P21: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P22: [asynchronous execution support for Custom Scan](https://commitfest.postgresql.org/patch/3813/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P23: [Issue in postgres_fdw causing unnecessary wait for cancel request reply](https://commitfest.postgresql.org/patch/4329/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0001-aio-Add-fsync-support.patch not read
- attachment limit (8) reached; v1-0002-Issue-SyncDataDirectory-fsyncs-through-AIO.patch not read
- attachment limit (8) reached; v1-0003-Issue-checkpointer-fsyncs-asynchronously.patch not read
- attachment limit (8) reached; v1-0004-Allow-IO-workers-to-execute-SLRU-fsyncs.patch not read
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 152 s; cost reported by the CLI: $2.50
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 98 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:07.097Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
