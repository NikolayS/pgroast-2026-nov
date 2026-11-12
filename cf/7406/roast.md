# Design pre-review: Reset unlogged relations before syncing the data directory?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7406](https://commitfest.postgresql.org/patch/7406/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWW_tx5ZacHLjCLHyvPkcjc%3Dy3OncLJJ4xyL7rwOFzGkkw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). The goal is clear and the cost falls on nobody who does not benefit, so levels 1a and 1b pass. One fixable level 2 objection survived, replication-ondisk-1: SyncDataDirectory() now runs after InitWalRecovery(), and the thread has no answer on why that is safe. The author asks this in message #1 and no committer has replied. A committer who knows the startup code has to confirm the ordering.

Not checked: The code was not read, so the exact position of SyncDataDirectory() relative to the steps inside InitWalRecovery() could not be checked. Message #4 was not shown.

## Prior attempts

- [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://www.postgresql.org/message-id/flat/7C1703E7-F3F3-43FA-86EB-177C671BF33C@vmware.com) (related; Returned with feedback, 2021-10-01). "Anyway, I think that it would be good to do more benchmarking for this patch first, and the patch you are proposing is enough for that." ([message](https://www.postgresql.org/message-id/YR88OzOxILlDtLVd%40paquier.xyz)) Different now: P4 skipped fsync of unchanged files in pg_rewind, a different tool and code path. This patch reorders unlogged cleanup and SyncDataDirectory() at crash startup. P4's request for more benchmarks is relevant because message #1 measured only without a cache flush.

## Objections a reviewer is likely to raise

### 1. InitWalRecovery() now runs before the data directory is synced

The server crashes and restarts. InitWalRecovery() reads the checkpoint record and backup_label from the OS page cache, restores tablespace links and durably renames tablespace_map. Power is lost before SyncDataDirectory() finishes, so the next start finds the renamed map but may not find links or WAL that were never flushed.

The cleanup can only run after the tablespace links exist, so the patch has to move SyncDataDirectory() after InitWalRecovery(). Today the sync runs first, so recovery setup only reads state that is already durable. The proposal does not list which reads and durable renames in InitWalRecovery() would now depend on unsynced files. In message #1 the author asks this same question, and no committer has answered it in messages #2-#4.

**Question:** Which files does InitWalRecovery() read or rename on a crash start, and why is each step safe if the OS loses unsynced data before SyncDataDirectory() completes?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAJTYsWW_tx5ZacHLjCLHyvPkcjc%3Dy3OncLJJ4xyL7rwOFzGkkw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/E22A0BD3-432F-4CF6-94FB-56717CA3E8FA%40gmail.com), [#4](https://www.postgresql.org/message-id/766EEB3A-4F42-4331-866A-4A38277522B6%40gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which files does InitWalRecovery() read or rename on a crash start, and why is each step safe if the OS loses unsynced data before SyncDataDirectory() completes?
2. What happens if the server crashes or loses power after the early cleanup but before SyncDataDirectory() finishes?
3. What are the patched and unpatched crash startup times when the OS page cache is flushed before restart?
4. Does a test cover a crash start with unlogged tables in a tablespace under the new ordering?

## Touch points

- **on-disk format / pg_upgrade:** The patch changes when unlogged relation files are removed relative to the crash-start data directory sync; no format change.
- **WAL / replication / standby:** The patch reorders InitWalRecovery(), unlogged cleanup and SyncDataDirectory() in the startup process in xlog.c.
- **maintenance burden ("who maintains this in 10 years"):** The unlogged cleanup now runs at two points chosen by start type, each with its own ordering reason.

## Who would commit it, and when

- Owner: silent. Someone who knows StartupXLOG, InitWalRecovery() and the crash-start fsync rules has to sign off; no committer has replied in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost low. The patch is about 4 KiB in one file and helps clusters with large unlogged tables on crash restart, with no user-visible surface. It waits on a committer who knows the startup code to answer the ordering question from message #1.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
