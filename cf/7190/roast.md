# Design pre-review: aio: Async fsyncs for crash recovery and checkpointer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7190](https://commitfest.postgresql.org/patch/7190/); [thread](https://www.postgresql.org/message-id/flat/CAN55FZ0vLWJQNB%3DHuHXG2wabFjXJd6OWTa3%3DkRzwObdZD9poHQ%40mail.gmail.com), 9 of 9 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal of overlapping checkpoint fsyncs, and all four are fixable. The largest risk is replication-ondisk-1: the new concurrent error path must still PANIC on any fsync failure, and no test injects one. already-exists-2 and maintenance-fit-2 ask for measurements at default checkpoint_flush_after and of foreground latency. already-exists-1 affects only the SyncDataDirectory() patch. Only 3 of 9 messages were visible.

Not checked: The code was not read, so the PANIC and retry rules in the 0006 commit message, completion ordering and restartpoint behaviour on standbys are unchecked. Messages #4 to #9 were seen only through the objections that cite them.

## Prior attempts

None found before 2026-08-25.

## Objections a reviewer is likely to raise

### 1. Checkpoint gain is measured with checkpoint_flush_after disabled

A Linux user runs the default checkpoint_flush_after, so writeback starts during the write phase and the final fsyncs find little dirty data. They upgrade and see a much smaller checkpoint speedup than the 2.7x to 3.9x reported. Meanwhile they take on new behaviour in sync.c and its error paths.

checkpoint_flush_after already exists to make the checkpoint's final fsyncs cheap. It does this by asking the kernel to start writeback early. Every checkpoint benchmark in message #1 sets checkpoint_flush_after = 0, which turns off that feature. So the numbers compare the patch against a configuration that most Linux installations do not use, and they do not show what the patch adds beyond the existing setting.

**Question:** What checkpoint speedup does the patch give over the default checkpoint_flush_after setting on Linux, for the same 4000-file workload?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAN55FZ0vLWJQNB%3DHuHXG2wabFjXJd6OWTa3%3DkRzwObdZD9poHQ%40mail.gmail.com); `already-exists-2`</sub>

### 2. Async SyncDataDirectory() duplicates recovery_init_sync_method=syncfs on Linux

A Linux user with slow crash-recovery startup sets io_method=io_uring and gets about the speed that recovery_init_sync_method=syncfs already gives. The project now keeps a new AIO path in fd.c, a PGAIO_TID_SYNC target, a new wait event and a doc paragraph, all for one case where an existing setting matches it.

io_uring is Linux-only, and syncfs is available on Linux. In message #5 the author measured syncfs as better than or equal to the AIO path, and in messages #6 and #8 both sides agreed the main benefit is the checkpoint. Under the worker method, patch 0004 runs these fsyncs synchronously anyway. The case it still helps is a data directory on a filesystem shared with other busy data, where syncfs also flushes unrelated files, and the thread has no measurement of that case.

**Question:** On which Linux setups is syncfs unsuitable but io_uring AIO fsync faster, and could patch 0004 be dropped from the series without hurting the checkpoint work?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#5](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/C24C12AB-DE89-4EC8-AF08-6157CDA9A8CA%40gmail.com), [#8](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com); `already-exists-1`</sub>

### 3. Concurrent fsync error paths change PANIC handling but are untested

During a checkpoint, one in-flight fsync fails with a non-PANIC error and the cleanup drains the others, one of which returns EIO. If that EIO is classed as a retryable failure, the next checkpoint's fsync succeeds on clean pages. The redo pointer then moves past WAL that the lost writes still need, and nothing reports it.

The v3-0006 commit message replaces the serial loop with new rules: preserve the first error that requires PANIC, leave other errors to the next checkpoint, and skip reinterpreting handled deletions during cleanup. A wrong outcome here repeats the loss case that made Postgres PANIC on fsync failure. The worker path adds a reopen-and-fsync in another process plus -errno passing (v3-0005), so the error reaches the checkpointer by a new route. Message #9 and the brief say no tests exist yet, and the thread runs no failing fsync, so the invariant is argued but not shown.

**Question:** Which fault-injection tests show that EIO from any in-flight fsync, under worker and io_uring, during a checkpoint or restartpoint, still leads to PANIC?

<sub>fixable, level 2, WAL / replication / standby; messages [#5](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAN55FZ2i-%3D87NWg8v66hbbm18o%2BkBmCjyLyX4fE7s49c9JbEBw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Checkpoint fsync bursts share the default worker pool with reads

An OLTP cluster runs the default io_method=worker. At each checkpoint's sync phase, up to io_max_concurrency fsyncs fill the worker queue and the device, and foreground reads queue behind them. Users see read latency spikes at every checkpoint and have no setting that caps fsync concurrency separately.

Yuhang raised storage bandwidth contention in message #4 and again in message #6. The author's answer in message #5 is to add more I/O workers, and message #6 replies that this does not address device bandwidth. The benchmarks run on an otherwise idle system with checkpoint_flush_after = 0. That setting moves writeback into the fsyncs and makes serial fsync look worst, so the cost to non-users and the gain under default flush settings both go unmeasured.

**Question:** What happens to foreground read latency during a checkpoint with default checkpoint_flush_after and io_method=worker, and how can an operator limit fsync concurrency?

<sub>fixable, level 2, performance on the hot path; messages [#4](https://www.postgresql.org/message-id/D707392D-96FF-4867-943D-564DAD6644B3%40gmail.com), [#5](https://www.postgresql.org/message-id/CAN55FZ2WnU9pMtjKP-%3DV%2B41gzPABuY6hXB_xO3CnOuMvhTVJvg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/C24C12AB-DE89-4EC8-AF08-6157CDA9A8CA%40gmail.com), [#8](https://www.postgresql.org/message-id/CAN55FZ2NT6529QhB_dcrZ5LaSL20aayhP33ZisZVFSg2hOqLXg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. What checkpoint speedup does the patch give with the default checkpoint_flush_after on Linux, for the same 4000-file workload?
2. Which fault-injection tests show that EIO from any in-flight fsync, under worker and io_uring, in a checkpoint or restartpoint, still leads to PANIC?
3. What happens to foreground read latency during a checkpoint with io_method=worker and default flush settings, on slower or shared storage?
4. How can an operator limit checkpoint fsync concurrency without also lowering io_max_concurrency for reads?
5. On which Linux setups is syncfs unsuitable but io_uring fsync faster, and can the SyncDataDirectory() patch be dropped without hurting the checkpoint work?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: Concurrent fsync error paths change PANIC handling but are untested
- **WAL / replication / standby:** Checkpoints and restartpoints advance the redo pointer only after these fsyncs succeed, and the patch rewrites that error path.
- **ABI / extension API:** The patch reshapes SyncOps, adds PGAIO_OP_FSYNC and two AIO targets, and lets reopen callbacks return -errno.
- **maintenance burden ("who maintains this in 10 years"):** sync.c gains in-flight bookkeeping, deferred completion and first-PANIC-wins rules that later changes must keep correct.
- **performance on the hot path:** Under the default io_method=worker, checkpoint fsyncs share the I/O worker pool and the device with foreground reads.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The SyncDataDirectory() conversion overlaps with recovery_init_sync_method=syncfs, which the author measured as equal or faster.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows both the AIO subsystem and the checkpointer's sync.c fsync error handling. The visible thread and the CommitFest entry show no committer taking part.
- Priority: proceed; audience a sizeable minority; review cost high. Clusters with many dirty files per checkpoint gain, but the size of the gain at default flush settings is not shown. The series is about 125 KiB across AIO, sync.c, smgr, SLRU and fd.c, and it changes durability error paths, so review and upkeep are costly. The thread is active and the open points are fixable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 5.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
