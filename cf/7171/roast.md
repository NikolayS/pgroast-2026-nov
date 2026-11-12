# Design pre-review: Write Combining

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7171](https://commitfest.postgresql.org/patch/7171/); [thread](https://www.postgresql.org/message-id/flat/CAAKRu_bcWRvRwZUop_d9vzF9nHAiT%2B-uPzkJ%3DS3ShZ1GqeAYOw%40mail.gmail.com), 43 of 43 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated and demand is shown, and no surviving objection is fatal. Level 2 is not yet settled. replication-ondisk-1 (in-place batch checksums under share-locked hint-bit updates) and replication-ondisk-2 (the recovery assert failure and untraced invalid-page warnings) are open safety points. standards-compat-1 and standards-compat-2 change the measured behaviour of checkpoint_completion_target and bgwriter_lru_maxpages. All are fixable inside this design, and they need someone who knows the buffer manager and recovery.

Not checked: The code was not read, so the v16 checksum path, the LSN handling of neighbor blocks in recovery and the cost of the neighbor search on the backend path could not be checked. Only 3 of 43 messages were visible; later messages are known through the objections.

## Prior attempts

- [Vectored I/O in bulk_write.c](https://www.postgresql.org/message-id/flat/CA+hUKGLx5bLwezZKAYB2O_qHj=ov10RpgRVY7e8TSJVE74oVjg@mail.gmail.com) (similar idea; Returned with feedback, 2024-10-07). "I'd like to try something more ambitious that streamifies this and also the "main" writeback paths." ([message](https://www.postgresql.org/message-id/CA%2BhUKGJRJk06OnLPV2GHmJg2vRx%2B4OXKr_5pFX3Dk%3DqMGXWtPw%40mail.gmail.com)) Different now: P2 combined writes only in bulk_write.c, and its author set it aside to try a broader design that also covers the main writeback paths. This proposal covers those paths (checkpointer, bgwriter, backends, bulkwrite ring), with synchronous writes and without streams.

## Objections a reviewer is likely to raise

### 1. Batch checksum on shared lock can write pages with stale checksums

On a cluster with data checksums, a query sets a hint bit under a share lock while a combined write holds the batch. The page reaches disk with a checksum computed before the hint change. A later read, pg_basebackup verification or a standby reports a checksum failure on a page that is not corrupt.

Message #24 says the batch path checksums pages in place, unlike master's PageSetChecksumCopy(), and calls the series uncommittable until separate hint-bit lock work lands. The material does not show that the v16 write path has removed this dependency. Every combined write in checkpointer, bgwriter and backends would then depend on work outside this proposal.

**Question:** Does v16 copy pages before checksumming them, and if not, which committed lock change makes in-place batch checksums safe under share-locked hint-bit updates?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#24](https://www.postgresql.org/message-id/CAAKRu_ZiEpE_EHww3S3-E3iznybdnX8mXSO7Wsuru7%3DP9Y%3DczQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. WAL-before-data rule for combined writes during recovery is not stated

A user restores a base backup and starts recovery with checksums off. The startup process writes a combined batch whose neighbor LSN is past what XLogFlush() can satisfy. An assert build crashes, and a production build writes the batch with only a warning, so nothing checks the rule there.

Message #43 shows the new WriteBuffers() assert failing in recovery, because XLogFlush() only updates minRecoveryPoint and warns there. Message #40 confirms that neighbors can raise max_lsn above what the required victim needs. Messages #27 and #30 report intermittent invalid-page errors in 027_stream_regress and a zeroed FSM page with no root cause found. The proposal does not state which invariant a combined write keeps on a standby, during crash recovery and at a restartpoint.

**Question:** What rule ties a combined batch's max_lsn to minRecoveryPoint on a standby and in crash recovery, and have the invalid-page failures from v13 and v14 been traced to a cause?

<sub>fixable, level 2, WAL / replication / standby; messages [#27](https://www.postgresql.org/message-id/CAAKRu_b_e96kWChReBPNjsD7GJuGKoM%2BexdyodP%2B8Oa%3D-T%3DpZA%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CAAKRu_Ygia_e2aVJcnkSksy85wuYwk_WattzwKJSJvcVj72Tuw%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/CABXr29EvZS4tC8K9stk8fcwhhC7Zy_ND0jDvreMuvepKntNiwg%40mail.gmail.com), [#43](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. Spread checkpoints no longer follow checkpoint_completion_target

A DBA sets checkpoint_completion_target = 0.9 to spread checkpoint writes over the interval. After upgrade, the write phase finishes in about a quarter of the expected time, so the I/O lands in a burst. The DBA sees foreground latency spikes at each checkpoint and has no setting to restore the old pacing.

CheckpointWriteDelay() is now called once per combined write and still sleeps at most 100 ms per call. In message #42 a 26.9 s write phase drops to 7.1 s, and message #43 reports 26.3 s dropping to 11.2 s. The documented meaning of checkpoint_completion_target changes for everyone who uses spread checkpoints, the default case, while the measured gains in message #33 come from immediate checkpoints and backend writes. Message #43 attaches a fix, but v16 has not adopted one.

**Question:** With combining on, does a spread checkpoint keep the write-phase duration that checkpoint_completion_target asks for, on the setups in messages #42 and #43?

<sub>fixable, level 2, design; messages [#33](https://www.postgresql.org/message-id/CAAKRu_ZoKTpnmQ3MmmQXTpG7msHrP3jNWC68sf8R82-8cgb7Kw%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/201425E8-D617-4971-BE7E-E94DDAF8356F%40yandex-team.ru), [#43](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com); `standards-compat-1`</sub>

### 4. bgwriter_lru_maxpages stops being a hard per-round write limit

An operator lowers bgwriter_lru_maxpages to cap background writes on shared or IOPS-limited storage. After upgrade, each round can write a whole combined run of up to io_combine_limit blocks, so the bgwriter exceeds the cap. The operator only finds out from pg_stat_io or storage throttling.

Message #42 shows that one call to run_bgwriter_cleaner(1) writes six contiguous buffers, and asks for the remaining budget to be passed into GatherContiguousDirtyBuffers(). Message #43 shows a second effect: neighbors use up the budget, so clock-hand cleaning in 60 s fell from 28000 to 20218 buffers. The proposal does not yet say whether the documented limit counts buffers or IOs. It also does not say whether neighbor writes count against it.

**Question:** Will bgwriter_lru_maxpages stay a strict per-round buffer limit, and if not, what wording and release note replace its documented meaning?

<sub>fixable, level 2, design; messages [#42](https://www.postgresql.org/message-id/201425E8-D617-4971-BE7E-E94DDAF8356F%40yandex-team.ru), [#43](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com); `standards-compat-2`</sub>

### 5. No separate control for foreground neighbor writes

A latency-sensitive OLTP site keeps a hot set resident and continuously redirtied while VACUUM cycles colder blocks. Each backend that needs a clean buffer also writes up to io_combine_limit of hot neighbors that are dirtied again soon. To stop this, the DBA can only lower io_combine_limit, which also shrinks read combining for sequential scans.

Message #41 describes this adverse case and has no answer yet. Message #37 removed eager_clean_max_batch_size, and message #39 confirms io_combine_limit now caps both reads and writes. The benchmarks in message #33 cover IOPS-limited storage, and message #40 covers WAL-flush cost on local NVMe. Neither measures redundant foreground writes of hot pages, so every backend carries the extra work and no setting turns it off without also losing read combining.

**Question:** How much write amplification does the hot-set-plus-VACUUM workload from message #41 show, and how would a DBA limit backend neighbor writes without changing read combining?

<sub>fixable, level 2, performance on the hot path; messages [#33](https://www.postgresql.org/message-id/CAAKRu_ZoKTpnmQ3MmmQXTpG7msHrP3jNWC68sf8R82-8cgb7Kw%40mail.gmail.com), [#37](https://www.postgresql.org/message-id/CAAKRu_Z%2BGEgkHqQPajsy0ApDWYTvfmR5G6AjnE7mVcy1_00kxQ%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CAAKRu_b%3DroA_nntHxV3POk06b0JPhT5N3HDDSNRBV%2B2FgLcUBQ%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/CABXr29EvZS4tC8K9stk8fcwhhC7Zy_ND0jDvreMuvepKntNiwg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/9FB15BAB-BD87-419E-B327-B63E6E443E08%40yandex-team.ru); `maintenance-fit-2`</sub>

## Questions for the author

1. Does v16 copy pages before checksumming them, and if not, which committed lock change makes in-place batch checksums safe against hint-bit updates under a share lock?
2. What rule ties a combined batch's highest LSN to minRecoveryPoint in recovery, and what caused the invalid-page warnings reported in messages #27 and #30?
3. With combining on, does a spread checkpoint keep the write-phase duration that checkpoint_completion_target asks for, on the setups in messages #42 and #43?
4. Does bgwriter_lru_maxpages stay a strict per-round buffer limit, and do neighbor writes count against it?
5. How much extra writing does the hot-set-plus-VACUUM workload from message #41 show, and can a DBA limit backend neighbor writes without lowering read combining?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: Batch checksum on shared lock can write pages with stale checksums
- **WAL / replication / standby:** Combined writes flush WAL up to the batch's highest LSN, including in recovery, where message #43 reports an assert failure.
- **ABI / extension API:** The series changes PinBuffer(), removes SyncOneBuffer() and BufferSync(), and edits bufpage.h and buf_internals.h.
- **new GUC:** No new GUC, but io_combine_limit now also caps writes, and bgwriter_lru_maxpages and checkpoint_completion_target behave differently.
- **backpatchability:** see objection standards-compat-1: Spread checkpoints no longer follow checkpoint_completion_target
- **maintenance burden ("who maintains this in 10 years"):** Checkpointer, bgwriter and backend write paths move to a batch write API; a documented dtrace probe is renamed.
- **performance on the hot path:** Backends that need a victim buffer now also look for and write dirty neighbor blocks.

## Who would commit it, and when

- Owner: qualified committer engaged. The panel notes that the author is an active committer in the buffer manager area. The owner must also know recovery, minRecoveryPoint handling and data checksums.
- Priority: fix design first; audience most users; review cost high. Message #33 reports about 30% more TPS on IOPS-limited storage, and direct IO needs combined writes, so the gain is broad. The series rewrites core write paths in bufmgr.c and the checkpointer, and four reviewer-measured design points are still open in v16.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 18.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
