# Design pre-review: Skip prefetch for block references that follow a FPW or WILL_INIT of the same block

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6730](https://commitfest.postgresql.org/patch/6730/); [thread](https://www.postgresql.org/message-id/flat/CAHg%2BQDf%3DeFFnOvmaPJQrXNiXaTOhhJhvO%3DDzmVUi%3DVGFRh5B0A%40mail.gmail.com), 17 of 17 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with measurements, the thread shows demand from several independent people, and the change reuses the existing recent-blocks window in one file. Prefetch is advice only, so a skipped prefetch cannot change replay results. The open points are scope decisions for a committer: back-patching, v3 versus the hash-table variant, and the planned prefetcher rework.

Not checked: Only 3 of 17 messages and the prose of v1 and v2 were read; v3 and the hash-table variant were not seen. The back-patch risk and the CPU cost of the added bookkeeping in replay cannot be judged without the code.

## Prior attempts

- [Revert recovery prefetching feature.](https://www.postgresql.org/message-id/flat/CAOuzzgrn7iKnFRsB4MHp3UisEQAGgZMbk_ViTN4HV4-Ksq8zCg@mail.gmail.com) (related; Reverted, 2021-05-10). "This set of commits has some bugs with known fixes, but at this late stage in the release cycle it seems best to revert and resubmit next time, along with some new automated test coverage for this whole area." Different now: P1 reverted the whole recovery prefetch feature before release because of late-cycle bugs and missing tests. This patch is a small change to the duplicate-detection window of the feature that went in later, and it does not change replay results.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Is the patch meant for back-patching, and what is the risk in stable branches if pg_stat_recovery_prefetch counts shift from prefetch to skip_rep?
2. What does v3 gain on a table with more than four indexes, where the window of XLOGPREFETCHER_SEQ_WINDOW_SIZE blocks left most prefetches in place?
3. Should the hash-table variant for HEAD replace v3 or follow it, and what replay CPU cost does it add?
4. How does this change fit with the planned AIO or read_stream rework of the prefetcher: does it survive that rework or only cover the releases before it?

## Touch points

- **WAL / replication / standby:** The patch changes which blocks the recovery prefetcher sends to posix_fadvise during standby replay and crash recovery.
- **backpatchability:** Two participants asked for a back-patch, and the patch does not state the risk for stable branches.
- **performance on the hot path:** The patch adds recent-blocks bookkeeping on the FPW and WILL_INIT skip paths of XLogPrefetcherNextBlock(), which runs for every block reference in replay.

## Who would commit it, and when

- Owner: qualified committer engaged. Thomas Munro, who knows xlogprefetcher.c, takes part in the thread and is studying the reports and an alternative idea.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is under 4 KiB in one file and adds no user-visible surface beyond shifted counter values. The thread reports a production standby problem and several independent reproductions, and the author measured about 31% less replay time on NVMe.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
