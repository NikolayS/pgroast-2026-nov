# Design pre-review: Write Combining

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7171](https://commitfest.postgresql.org/patch/7171/) (Needs review); authors: Melanie Plageman (melanieplageman)
- Thread: [Checkpointer write combining](https://www.postgresql.org/message-id/flat/CAAKRu_bcWRvRwZUop_d9vzF9nHAiT%2B-uPzkJ%3DS3ShZ1GqeAYOw%40mail.gmail.com), started 2025-09-02
- Messages read: 43 of 43
- Patch prose read: v16-0001-Allow-PinBuffer-to-skip-increasing-usage-count.patch, v16-0002-Add-database-Oid-to-CkptSortItem.patch, v16-0003-Inline-BufferSync-into-CheckPointBuffers.patch, v16-0004-Write-combining-for-checkpointer.patch, v16-0005-Add-checkpointer-write-combining-test.patch, v16-0006-Remove-SyncOneBuffer-and-refactor-BgBufferSync.patch, v16-0007-pg_stat_io-Count-buffer-eviction-after-invalidat.patch, v16-0008-Call-BufferLockUnlock-directly-where-the-buffer-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated and demand is shown, and no surviving objection is fatal. Level 2 is not yet settled. replication-ondisk-1 (in-place batch checksums under share-locked hint-bit updates) and replication-ondisk-2 (the recovery assert failure and untraced invalid-page warnings) are open safety points. standards-compat-1 and standards-compat-2 change the measured behaviour of checkpoint_completion_target and bgwriter_lru_maxpages. All are fixable inside this design, and they need someone who knows the buffer manager and recovery.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the v16 checksum path, the LSN handling of neighbor blocks in recovery and the cost of the neighbor search on the backend path could not be checked. Only 3 of 43 messages were visible; later messages are known through the objections.

## The proposal, restated

**Problem.** Postgres writes dirty shared buffers one block at a time, so checkpoints and buffer cleaning use many small write IOs. This limits write throughput, most of all on IOPS-limited storage. The author also says all writers will need write combining before Postgres can use direct IO.

**Proposed design.** The checkpointer, bgwriter, client backends and the bulkwrite strategy ring write runs of contiguous dirty blocks as one vectored write of up to io_combine_limit. Backends and bgwriter look for dirty blocks before and after the victim block. The series also refactors victim buffer selection and adds a WriteBuffersOperation object shaped for later AIO writes.

**User-visible surface.**
- dtrace probe buffer__sync__written renamed to buffers__sync__written; it now reports buffers per combined IO, not a buffer ID.
- pg_stat_io: evictions and reuses counted once when invalidating a victim buffer fails; backends and bgwriter now show larger writes.
- io_combine_limit now also caps write size, not only read size.
- test_aio module gains functions to dirty, flush and invalidate specific relation blocks.

**What the author claims.**
- Immediate checkpoints are at least 20% faster (about 6.5 s to 5 s, 16 GiB shared_buffers, local SSD).
- pgbench scale 500 on IOPS-limited storage: about 30% more TPS and 75% better p99 latency; almost 70% more TPS with synchronous_commit off.
- Most of the gain comes from client backends combining their own writes; checkpointer combining adds little there because the kernel already combines its writes.
- The API shape will carry over to AIO writes with separate start, completion and wait phases.

**Thread state.** Andrey Borodin, Chao Li, Haibo Yan and shihao zhong have reviewed v16; open items include spread checkpoints finishing early, bgwriter_lru_maxpages being exceeded, and an assert failure during recovery from a base backup. The checksum race with hint bits waits on separate work by Andres, and the author reported an intermittent invalid-page warning in v14.

## 2. Prior attempts

### Vectored I/O in bulk_write.c (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4891/](https://commitfest.postgresql.org/patch/4891/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKGLx5bLwezZKAYB2O_qHj=ov10RpgRVY7e8TSJVE74oVjg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGLx5bLwezZKAYB2O_qHj=ov10RpgRVY7e8TSJVE74oVjg@mail.gmail.com)
- **Date:** decided 2024-10-07; started 2024-03-10
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author decided not to commit it for v17. He chose to try a broader design that would use streaming writes for this path and for the main writeback paths. Andres Freund's objection to merging smgrzeroextend() into the same interface was settled by dropping that part, so it did not decide the outcome. Noah Misch later asked whether the CF entry was obsolete, and the visible messages show no reply.
- **Reason (quote):** "I'd like to try something more ambitious that streamifies this and also the "main" writeback paths." ([message](https://www.postgresql.org/message-id/CA%2BhUKGJRJk06OnLPV2GHmJg2vRx%2B4OXKr_5pFX3Dk%3DqMGXWtPw%40mail.gmail.com))
- **What is different this time:** P2 combined writes only in bulk_write.c, and its author set it aside to try a broader design that also covers the main writeback paths. This proposal covers those paths (checkpointer, bgwriter, backends, bulkwrite ring), with synchronous writes and without streams.


## 3. Objections a reviewer is likely to raise

### 1. Batch checksum on shared lock can write pages with stale checksums

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-1`

On a cluster with data checksums, a query sets a hint bit under a share lock while a combined write holds the batch. The page reaches disk with a checksum computed before the hint change. A later read, pg_basebackup verification or a standby reports a checksum failure on a page that is not corrupt.

Message #24 says the batch path checksums pages in place, unlike master's PageSetChecksumCopy(), and calls the series uncommittable until separate hint-bit lock work lands. The material does not show that the v16 write path has removed this dependency. Every combined write in checkpointer, bgwriter and backends would then depend on work outside this proposal.

Thread: [#24 (Melanie Plageman, 2025-11-20)](https://www.postgresql.org/message-id/CAAKRu_ZiEpE_EHww3S3-E3iznybdnX8mXSO7Wsuru7%3DP9Y%3DczQ%40mail.gmail.com)

**Question:** Does v16 copy pages before checksumming them, and if not, which committed lock change makes in-place batch checksums safe under share-locked hint-bit updates?

Counter-check: held. Message #24 says in-place batch checksums are not committable until the hint-bit lock work lands. Nothing in the visible v16 prose says that the dependency is met or that pages are copied before checksumming.

### 2. WAL-before-data rule for combined writes during recovery is not stated

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-2`

A user restores a base backup and starts recovery with checksums off. The startup process writes a combined batch whose neighbor LSN is past what XLogFlush() can satisfy. An assert build crashes, and a production build writes the batch with only a warning, so nothing checks the rule there.

Message #43 shows the new WriteBuffers() assert failing in recovery, because XLogFlush() only updates minRecoveryPoint and warns there. Message #40 confirms that neighbors can raise max_lsn above what the required victim needs. Messages #27 and #30 report intermittent invalid-page errors in 027_stream_regress and a zeroed FSM page with no root cause found. The proposal does not state which invariant a combined write keeps on a standby, during crash recovery and at a restartpoint.

Thread: [#27 (Melanie Plageman, 2026-01-14)](https://www.postgresql.org/message-id/CAAKRu_b_e96kWChReBPNjsD7GJuGKoM%2BexdyodP%2B8Oa%3D-T%3DpZA%40mail.gmail.com), [#30 (Melanie Plageman, 2026-02-19)](https://www.postgresql.org/message-id/CAAKRu_Ygia_e2aVJcnkSksy85wuYwk_WattzwKJSJvcVj72Tuw%40mail.gmail.com), [#40 (Haibo Yan, 2026-08-21)](https://www.postgresql.org/message-id/CABXr29EvZS4tC8K9stk8fcwhhC7Zy_ND0jDvreMuvepKntNiwg%40mail.gmail.com), [#43 (shihao zhong, 2026-10-04)](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com)

**Question:** What rule ties a combined batch's max_lsn to minRecoveryPoint on a standby and in crash recovery, and have the invalid-page failures from v13 and v14 been traced to a cause?

Counter-check: held. The assert failure in message #43 and the untraced invalid-page warnings in messages #27 and #30 are open. The production-build part is overstated, because master also only warns in that case.

### 3. Spread checkpoints no longer follow checkpoint_completion_target

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

A DBA sets checkpoint_completion_target = 0.9 to spread checkpoint writes over the interval. After upgrade, the write phase finishes in about a quarter of the expected time, so the I/O lands in a burst. The DBA sees foreground latency spikes at each checkpoint and has no setting to restore the old pacing.

CheckpointWriteDelay() is now called once per combined write and still sleeps at most 100 ms per call. In message #42 a 26.9 s write phase drops to 7.1 s, and message #43 reports 26.3 s dropping to 11.2 s. The documented meaning of checkpoint_completion_target changes for everyone who uses spread checkpoints, the default case, while the measured gains in message #33 come from immediate checkpoints and backend writes. Message #43 attaches a fix, but v16 has not adopted one.

Thread: [#33 (Melanie Plageman, 2026-06-19)](https://www.postgresql.org/message-id/CAAKRu_ZoKTpnmQ3MmmQXTpG7msHrP3jNWC68sf8R82-8cgb7Kw%40mail.gmail.com), [#42 (Andrey Borodin, 2026-09-14)](https://www.postgresql.org/message-id/201425E8-D617-4971-BE7E-E94DDAF8356F%40yandex-team.ru), [#43 (shihao zhong, 2026-10-04)](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com)

**Question:** With combining on, does a spread checkpoint keep the write-phase duration that checkpoint_completion_target asks for, on the setups in messages #42 and #43?

Counter-check: held. Messages #42 and #43 each measure the shorter write phase, and v16 has not adopted a fix. One detail is wrong: message #42 shows io_combine_limit = 8kB restores the old pacing, so a setting does exist.

### 4. bgwriter_lru_maxpages stops being a hard per-round write limit

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An operator lowers bgwriter_lru_maxpages to cap background writes on shared or IOPS-limited storage. After upgrade, each round can write a whole combined run of up to io_combine_limit blocks, so the bgwriter exceeds the cap. The operator only finds out from pg_stat_io or storage throttling.

Message #42 shows that one call to run_bgwriter_cleaner(1) writes six contiguous buffers, and asks for the remaining budget to be passed into GatherContiguousDirtyBuffers(). Message #43 shows a second effect: neighbors use up the budget, so clock-hand cleaning in 60 s fell from 28000 to 20218 buffers. The proposal does not yet say whether the documented limit counts buffers or IOs. It also does not say whether neighbor writes count against it.

Thread: [#42 (Andrey Borodin, 2026-09-14)](https://www.postgresql.org/message-id/201425E8-D617-4971-BE7E-E94DDAF8356F%40yandex-team.ru), [#43 (shihao zhong, 2026-10-04)](https://www.postgresql.org/message-id/CAGRkXqRqJq%2B_wmyzbD5%2BHOQ4C4DPQ6f68d00qaVJzXHq5%2BiEbQ%40mail.gmail.com)

**Question:** Will bgwriter_lru_maxpages stay a strict per-round buffer limit, and if not, what wording and release note replace its documented meaning?

Counter-check: held. Message #42 shows one cleaner round writing six buffers past the limit, and message #43 shows fewer clock-hand buffers cleaned. The thread has no reply yet on how neighbors count against bgwriter_lru_maxpages.

### 5. No separate control for foreground neighbor writes

fixable · level 2, rubric item 4 · performance on the hot path, new GUC · `maintenance-fit-2`

A latency-sensitive OLTP site keeps a hot set resident and continuously redirtied while VACUUM cycles colder blocks. Each backend that needs a clean buffer also writes up to io_combine_limit of hot neighbors that are dirtied again soon. To stop this, the DBA can only lower io_combine_limit, which also shrinks read combining for sequential scans.

Message #41 describes this adverse case and has no answer yet. Message #37 removed eager_clean_max_batch_size, and message #39 confirms io_combine_limit now caps both reads and writes. The benchmarks in message #33 cover IOPS-limited storage, and message #40 covers WAL-flush cost on local NVMe. Neither measures redundant foreground writes of hot pages, so every backend carries the extra work and no setting turns it off without also losing read combining.

Thread: [#33 (Melanie Plageman, 2026-06-19)](https://www.postgresql.org/message-id/CAAKRu_ZoKTpnmQ3MmmQXTpG7msHrP3jNWC68sf8R82-8cgb7Kw%40mail.gmail.com), [#37 (Melanie Plageman, 2026-08-19)](https://www.postgresql.org/message-id/CAAKRu_Z%2BGEgkHqQPajsy0ApDWYTvfmR5G6AjnE7mVcy1_00kxQ%40mail.gmail.com), [#39 (Melanie Plageman, 2026-08-19)](https://www.postgresql.org/message-id/CAAKRu_b%3DroA_nntHxV3POk06b0JPhT5N3HDDSNRBV%2B2FgLcUBQ%40mail.gmail.com), [#40 (Haibo Yan, 2026-08-21)](https://www.postgresql.org/message-id/CABXr29EvZS4tC8K9stk8fcwhhC7Zy_ND0jDvreMuvepKntNiwg%40mail.gmail.com), [#41 (Andrey Borodin, 2026-08-24)](https://www.postgresql.org/message-id/9FB15BAB-BD87-419E-B327-B63E6E443E08%40yandex-team.ru)

**Question:** How much write amplification does the hot-set-plus-VACUUM workload from message #41 show, and how would a DBA limit backend neighbor writes without changing read combining?

Counter-check: held. Message #41 describes the hot-set case and the visible thread has no measurement of it; the pgbench runs in message #33 do not model a resident hot set. The request for a measurement stands even if a separate setting is not the right fix.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: Batch checksum on shared lock can write pages with stale checksums |
| WAL / replication / standby | **yes** | Combined writes flush WAL up to the batch's highest LSN, including in recovery, where message #43 reports an assert failure. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series changes PinBuffer(), removes SyncOneBuffer() and BufferSync(), and edits bufpage.h and buf_internals.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but io_combine_limit now also caps writes, and bgwriter_lru_maxpages and checkpoint_completion_target behave differently. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Spread checkpoints no longer follow checkpoint_completion_target |
| maintenance burden ("who maintains this in 10 years") | **yes** | Checkpointer, bgwriter and backend write paths move to a batch write API; a documented dtrace probe is renamed. |
| performance on the hot path | **yes** | Backends that need a victim buffer now also look for and write dirty neighbor blocks. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v16 copy pages before checksumming them, and if not, which committed lock change makes in-place batch checksums safe against hint-bit updates under a share lock?
2. What rule ties a combined batch's highest LSN to minRecoveryPoint in recovery, and what caused the invalid-page warnings reported in messages #27 and #30?
3. With combining on, does a spread checkpoint keep the write-phase duration that checkpoint_completion_target asks for, on the setups in messages #42 and #43?
4. Does bgwriter_lru_maxpages stay a strict per-round buffer limit, and do neighbor writes count against it?
5. How much extra writing does the hot-set-plus-VACUUM workload from message #41 show, and can a DBA limit backend neighbor writes without lowering read combining?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The panel notes that the author is an active committer in the buffer manager area. The owner must also know recovery, minRecoveryPoint handling and data checksums.
- **Liability:** The owner signs for the WAL-before-data rule and checksum correctness on every data write path, in normal running and in recovery, with no end date. The owner also signs for changed pacing of spread checkpoints and bgwriter rounds on existing installations.
- **Author follow-up:** Strong sign. The series reached v16 over about 14 months, and the author posted rebases and revisions in response to reviews.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** most users
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Message #33 reports about 30% more TPS on IOPS-limited storage, and direct IO needs combined writes, so the gain is broad. The series rewrites core write paths in bufmgr.c and the checkpointer, and four reviewer-measured design points are still open in v16.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` Renamed dtrace probe silently breaks existing tracing scripts. Refuted (not specific): The commit message for 0004 states the rename and gives the reason for the new argument. What remains is a monitoring.sgml update, already raised in message #43, and a release note; both are documentation work outside levels 1-2 and apply to any user-visible change.
- `maintenance-fit-1` Buffer-counted limits silently change meaning under combined writes. Refuted (not specific): Duplicate of standards-compat-1 and standards-compat-2, which cover the same two measured mismatches. The added point about future write paths names no concrete case.

### Review angles

- prior-attempts: 0 objection(s). The only close precedent, P2, was set aside in favour of a broader design that would also cover the main writeback paths. This proposal is that broader scope, so the P2 reason does not count against it. P7 is about checkpoint pacing under full_page_writes and has no recorded reason, so it is not used here. The open problems in the thread, such as spread checkpoint pacing (messages #42, #43) and bgwriter_lru_maxpages, are new design issues for other angles.
- standards-compat: 3 objection(s). No SQL syntax, catalog or pg_dump changes are proposed. The objections cover changed meaning of existing settings and of a documented probe.
- replication-ondisk: 2 objection(s). There is no on-disk format, catalog, WAL record or protocol change, so pg_upgrade and mixed-version replication are not affected. Both objections concern write-path invariants only.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how the checkpointer, bgwriter and backends write dirty shared buffers. It adds no SQL-callable function, privilege, role or GUC, and no new monitoring output beyond larger pg_stat_io write counts. The test_aio functions that dirty, flush and invalidate blocks are in a test module, not an installed extension. Data-integrity risks such as the hint-bit checksum race, the recovery assert and the invalid-page warning belong to the safety angle, not this one.
- maintenance-fit: 2 objection(s). Not applicable. Fit and ownership look sound: an active committer in the bufmgr area wrote the series, several reviewers are engaged, and the preparatory patches can be committed on their own.
- already-exists: 0 objection(s). No existing feature, setting or extension does this job. The *_flush_after settings only send writeback hints for writes already issued one block at a time, and kernel merging does not apply under direct IO. The proposal extends io_combine_limit, the existing read-combining limit, so it adds no second mechanism, and buffer manager internals cannot be replaced from an extension.
- demand: 0 objection(s). Demand is shown in the material. Write throughput on IOPS-limited storage is a broad, common problem, and the author reports about 30% more TPS and 75% better p99 latency in pgbench (message #33). Independent testers saw faster checkpoints (messages #3, #28), and direct IO, a project-wide goal, needs combined writes. The benefit goes to all write-heavy users. The open costs (spread checkpoint pacing, bgwriter_lru_maxpages) are design questions for other angles.

### Precedents retrieved

- P1: [teach pg_combinebackup to detect deleted files where possible](https://commitfest.postgresql.org/patch/4951/) (cf, Withdrawn, 2024-08-21) - judged not relevant; reason summarised from its thread
- P2: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P3: [Aggregate State Combine Support](https://commitfest.postgresql.org/patch/131/) (cf, Returned with feedback, 2015-07-27) - judged not relevant
- P4: [document that psql's FETCH_COUNT does not work with combined (\;) queries](https://commitfest.postgresql.org/patch/2219/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P5: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - judged not relevant
- P6: [Revert "Rename unicode_combining_table to unicode_width_table"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1563ecbc1be8b8e5c57651cf5c87f90dea9aea8f) (revert, Reverted, 2021-08-26) - judged not relevant
- P7: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - used in the review
- P8: [Batch (pipelined) mode for libpq](https://commitfest.postgresql.org/patch/634/) (cf, Returned with feedback, 2016-11-22) - judged not relevant
- P9: [bloom filter in hash joins with batching](https://commitfest.postgresql.org/patch/446/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P10: [Batch processing of RI checks](https://commitfest.postgresql.org/patch/2581/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P11: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P12: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P14: [hashjoin - gracefully increasing NTUP_PER_BUCKET instead of batching](https://commitfest.postgresql.org/patch/28/) (cf, Rejected, 2015-02-13) - judged not relevant; reason summarised from its thread
- P15: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P16: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P17: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant
- P18: [Reduce wakeup on idle for bgwriter & walwriter for >5s](https://commitfest.postgresql.org/patch/4029/) (cf, Withdrawn, 2023-03-30) - judged not relevant
- P19: [Revert "Add eager and lazy freezing strategies to VACUUM."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c6b4972664491de408670354fdbccc7c83a1c89) (revert, Reverted, 2023-01-26) - judged not relevant
- P20: [Start WAL receiver eagerly](https://commitfest.postgresql.org/patch/3376/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P21: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P22: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant
- P23: [psql - sort by size](https://commitfest.postgresql.org/patch/2188/) (cf, Withdrawn, 2019-09-13) - judged not relevant
- P24: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant

### Warnings

- attachment limit (8) reached; v16-0009-Make-StrategyRejectBuffer-encapsulate-more-logic.patch not read
- attachment limit (8) reached; v16-0010-Split-StrategyGetBuffer-into-source-specific-hel.patch not read
- attachment limit (8) reached; v16-0011-Introduce-ClaimVictimBuffer-helper.patch not read
- attachment limit (8) reached; v16-0012-Don-t-retry-the-strategy-ring-after-a-failed-clo.patch not read
- attachment limit (8) reached; v16-0013-Write-combining-for-client-backends.patch not read
- attachment limit (8) reached; v16-0014-Add-client-backend-write-combining-test.patch not read
- attachment limit (8) reached; v16-0015-Add-COPY-FROM-write-combining-test.patch not read
- attachment limit (8) reached; v16-0016-Add-VACUUM-write-combining-test.patch not read
- attachment limit (8) reached; v16-0017-Add-bgwriter-helper-to-write-buffers.patch not read
- attachment limit (8) reached; v16-0018-Write-combining-for-background-writer.patch not read
- attachment limit (8) reached; v16-0019-Add-bgwriter-write-combining-test.patch not read
- attachment limit (8) reached; v1-0005-Fix-XLogNeedsFlush-for-checkpointer.patch not read
- attachment limit (8) reached; v1-0001-Refactor-goto-into-for-loop-in-GetVictimBuffer.patch not read
- attachment limit (8) reached; v1-0004-Write-combining-for-BAS_BULKWRITE.patch not read
- attachment limit (8) reached; v1-0003-Eagerly-flush-bulkwrite-strategy-ring.patch not read
- attachment limit (8) reached; v1-0002-Split-FlushBuffer-into-two-parts.patch not read
- attachment limit (8) reached; v1-0006-Add-database-Oid-to-CkptSortItem.patch not read
- attachment limit (8) reached; v1-0007-Implement-checkpointer-data-write-combining.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 143 s; cost reported by the CLI: $5.43
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:25:03.590Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
