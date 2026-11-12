# Design pre-review: improving handling of FPIs during recovery prefetching

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7512](https://commitfest.postgresql.org/patch/7512/) (Needs review); authors: Tomas Vondra (fuzzycz)
- Thread: [improving handling of FPIs during recovery prefetching](https://www.postgresql.org/message-id/flat/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me), started 2026-10-26
- Messages read: 8 of 8
- Patch prose read: v1-0001-Avoid-useless-prefetches-in-case-of-recent-FPI-WA.patch, v1-0002-Separate-the-FPIs-from-recently-prefetched-blocks.patch, v1-0003-Rework-the-FPI-cache-as-a-set-associative-cache.patch, v1-0004-Move-FPI-tracking-into-separate-functions.patch, v1-0005-Use-a-proper-hash-function-for-FPI-history.patch, v1-0006-Ignore-replayed-FPIs-in-prefetch-FPI-history.patch, v1-0007-Avoid-duplicate-entries-in-FPI-history.patch, v1-0008-Size-recovery-FPI-history-by-the-look-ahead-dista.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the basic design. Three fixable level 2 points remain: maintenance-fit-2 (no measurement of the lookup cost for replay that gains nothing), replication-ondisk-1 (the squashed commit ties the fix to a catalog change, so it cannot be back-patched as is), and standards-compat-2 (the skip_fpi name). Message #1 shows demand from a field report and an earlier patch. Confidence is medium because the code was not read and a simpler alternative is expected but not posted.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the correctness of the eviction by replaying_lsn, the reset paths and the group overflow behaviour could not be checked. The expected simpler alternative is not posted, so no comparison was possible.

## The proposal, restated

**Problem.** recovery_prefetch still issues prefetches for a block after an earlier WAL record carries its full page image, though replaying the FPI installs the page with no read. On replicas with IOPS limits this unneeded I/O inflates replay lag, and it hurts sync replication when the walreceiver reconnects and the startup process must first apply the backlog.

**Proposed design.** The patch adds a separate history of pending FPIs in the WAL prefetcher, a set-associative cache keyed by a hash of the block. Entries whose LSN is before replaying_lsn count as empty, and the cache is sized from the look-ahead distance (maintenance_io_concurrency times a multiplier). The prefetcher skips blocks found in the history; the nine patches would be squashed into one commit.

**User-visible surface.**
- New bigint column skip_fpi in the pg_stat_recovery_prefetch view, documented in monitoring.sgml; the author calls the name provisional and easy to confuse with skip_fpw.

**What the author claims.**
- The reproducer UPDATE touches every page, and the standby then prefetches pages that the FPI replay will overwrite.
- The blog's 16x IOPS amplification comes from 128K read-ahead on the primary versus 8K reads on the standby.
- The FPI history uses at most about 400kB at maintenance_io_concurrency=1000, and there is only one per instance.
- The author could not build a case where the history lookup adds measurable CPU overhead.

**Thread state.** Jan Nidzwetzki confirmed check-world passes and that posix_fadvise() calls dropped from 23,502 to 821 in a crash recovery test; Shihao Zhong posted a TAP test, now at v3 and approved by Jan. Tomas then said Thomas Munro may post a simpler alternative, so review of this series is on hold.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Lookup cost on every block reference is unmeasured

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-2`

A standby that replays mostly non-FPI records, with prefetch already working well, now does a hash and a group scan for each block reference whenever any FPI is pending. If that cost adds up, replay gets slower on standbys that never had the IOPS problem. The project finds out from user reports after release.

In message #6 the author asks others to check the CPU cost for normal cases and says he could not build a bad case himself. Jan's numbers in message #3 only count posix_fadvise() calls on the workload that gains from the patch. Patch 0009 exits early only when no FPIs are pending, which is rare right after a checkpoint, and the history can grow to about 400kB at maintenance_io_concurrency=1000.

Thread: [#3 (Jan Nidzwetzki, 2026-11-06)](https://www.postgresql.org/message-id/e1b36cd3-c3a5-4da0-a1d7-497296d5d9f3%40planetscale.com), [#6 (Tomas Vondra, 2026-11-10)](https://www.postgresql.org/message-id/60a1221b-125b-481c-b019-66366a9e2673%40vondra.me)

**Question:** How much does replay throughput change on a CPU-bound standby with few useful skips, at default and high maintenance_io_concurrency?

Counter-check: held. In message #6 the author asks for this measurement and says nobody has done it. Jan's numbers count posix_fadvise() calls only on the workload that gains from the patch.

### 2. Stats column ties an outage fix to master only

fixable · level 2, rubric item 6 · backpatchability, catalog changes · `replication-ondisk-1`

An operator on Postgres 15 to 19 runs a replica under an IOPS limit and keeps seeing replay lag after a walreceiver restart. The squashed commit changes pg_proc.dat and system_views.sql, which needs a catversion bump, so it cannot go to back branches as one commit. The operator finds out only when the fix ships in a future major release.

Message #1 links recovery_prefetch to outages, but the only stated way out on released branches is recovery_prefetch = off. Patch 0002 mixes the behaviour change with the skip_fpi column in pg_stat_recovery_prefetch and a changed function in pg_proc.dat, and the plan squashes all nine patches into one commit. A committer would likely ask for the in-memory FPI history to be split from the catalog-visible counter, so the history alone can be backpatched.

Thread: [#1 (Tomas Vondra, 2026-10-26)](https://www.postgresql.org/message-id/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me)

**Question:** Is the FPI history meant for back branches, and if so, can it ship without the skip_fpi column and the pg_proc.dat change?

Counter-check: held. The thread does not say whether a back-patch is intended, and a pg_proc.dat change cannot go to stable branches. Performance fixes are often master-only, so this stands as a question about patch split and has no bearing on the design.

### 3. skip_fpi sits next to skip_fpw with a confusingly similar meaning

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax, catalog changes · `standards-compat-2`

A DBA reading pg_stat_recovery_prefetch sees skip_fpw and skip_fpi side by side and sums them, or graphs the wrong one. After release, monitoring tools hard-code the column name, and renaming it later breaks their queries.

The 0002 commit message says the name "is not very self-explanatory" and "easy to confuse with skip_fpw", and nothing in the thread picks a better one. Both columns count blocks skipped because of a full page image, one for the record that carries it and one for later records. View column names become part of the monitoring contract once released, so a committer would likely ask for the name and the doc text to be settled before commit.

Thread: [#1 (Tomas Vondra, 2026-10-26)](https://www.postgresql.org/message-id/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me)

**Question:** What column name and documentation wording make the difference from skip_fpw clear to someone who reads only the view?

Counter-check: held. The 0002 commit message itself says the name is easy to confuse with skip_fpw, and no message in the thread settles a name. Acknowledging the problem does not resolve it, and a rename before commit is cheap.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes which blocks the WAL prefetcher reads ahead during recovery on standbys and in crash recovery. |
| wire protocol | no |  |
| catalog changes | **yes** | 0002 changes pg_proc.dat and system_views.sql to add skip_fpi to pg_stat_recovery_prefetch. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: skip_fpi sits next to skip_fpw with a confusingly similar meaning |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 links the behaviour to outages on released branches, and the squashed commit includes a catalog change. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a second cache in xlogprefetcher.c with its own sizing and reset rules, plus a new view column. |
| performance on the hot path | **yes** | The prefetcher does a hash and a group scan for each block reference while any FPI is pending. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How much does replay throughput change on a CPU-bound standby with few useful skips, at default and at high maintenance_io_concurrency?
2. Is the FPI history meant for back branches, and if so, can it ship without the skip_fpi column and the pg_proc.dat change?
3. What column name and documentation wording make the difference from skip_fpw clear to someone who reads only the view?
4. Should the history also track BKPBLOCK_WILL_INIT blocks, as message #2 suggests, since redo reads no page in that case either?
5. Will problem (2), the slow restart of streaming after a walreceiver exit, be handled in a separate thread?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The author, Tomas Vondra, says in message #1 that he was involved in getting recovery_prefetch into PG 15. The owner must know the WAL prefetcher and the redo read path.
- **Liability:** The owner signs for replay speed on every standby that uses recovery_prefetch, and for the meaning of the skip_fpi column once monitoring tools read it. A wrong skip costs a synchronous read and does not change replayed data.
- **Author follow-up:** The author wrote a nine-patch series within days of the field report and says he will hold review for a possible simpler alternative, which suggests he stays engaged.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Replicas under IOPS limits gain, and message #3 shows posix_fadvise() calls dropping from 23,502 to 821. The change is mostly in xlogprefetcher.c plus one view column; review cost is set to medium because no code was read and a simpler alternative may replace the series.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` New skip_fpi column ties a likely back-patched fix to a catalog change. Refuted (not specific): Duplicate of replication-ondisk-1, which makes the same back-patch point and adds the concrete request to split the FPI history from the counter.
- `maintenance-fit-1` New skip_fpi view column blocks a back-branch fix and fixes the counter's meaning early. Refuted (not specific): Duplicate of replication-ondisk-1. The added point about the counter's meaning is weak, because the documented meaning of skip_fpi does not depend on how the prefetcher finds the pending FPI.
- `maintenance-fit-3` Second skip cache next to recent_block, while a simpler design is expected. Refuted (already handled): 0002 explains why the FPI history is separate from recent_block. 0006 and 0008 reset it whenever the prefetcher is reconfigured, including after XLogPrefetcherBeginRead(), so a future repositioning caller is covered, and the expected alternative is a timing matter the author already raised.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P10] and [P13] target the walreceiver restart problem, and [P18] adds fadvise to replay; none of their recorded reasons applies to FPI skipping. Message #1 cites a 2021 mention of FPIs, deferred as acceptable for v1, and a 2025 patch using the 4-entry recent_block cache, which this series includes as 0001 and extends.
- standards-compat: 2 objection(s). The only user-visible change is one view column; no syntax, GUC, pg_dump or client behaviour changes, so the standards angle sees only the two fixable points above.
- replication-ondisk: 1 objection(s). The FPI history lives only in the startup process's memory. The proposal changes no WAL record, page format or wire message, and the history is reset at reconfiguration and timeline switch. A wrongly skipped prefetch costs a synchronous read and does not corrupt data. Beyond the backpatch split, this angle finds no persistent or cross-node state to object to.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The FPI history lives only in the startup process, and its memory is bounded by maintenance_io_concurrency, which ordinary users cannot change for that process. A wrong or missing entry only adds or skips an advisory prefetch and never changes the data that is replayed. The new skip_fpi counter in pg_stat_recovery_prefetch shows no more than the counters the view already has.
- maintenance-fit: 3 objection(s). Two committers who know the area are engaged, so ownership is not a concern. The objections are about splitting the patch and showing measurements.
- already-exists: 0 objection(s). No existing feature does this job. Setting recovery_prefetch = off avoids the extra reads but drops all prefetching, and the 4-entry recent_block cache is too small for FPIs, as message #1 explains. Thomas Munro's simpler alternative (message #6) is not posted, so it cannot be compared.
- demand: 0 objection(s). The material shows real demand from independent sources: a public field report of outages tied to recovery_prefetch (message #1), an earlier report with its own patch about 18 months before, and the same concern raised in 2021. Jan's crash recovery test cut posix_fadvise() calls from 23,502 to 821 (message #3), and the user-visible cost is one stats column, so a demand objection would not hold.

### Precedents retrieved

- P1: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P2: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P3: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P4: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P5: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P6: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P7: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant; reason summarised from its thread
- P8: [Compensate for full_page_writes when spreading checkpoint I/O](https://commitfest.postgresql.org/patch/283/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P9: [Memory prefetching while sequentially fetching from SortTuple array, tuplestore](https://commitfest.postgresql.org/patch/305/) (cf, Returned with feedback, 2015-12-13) - judged not relevant
- P10: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - used in the review; reason summarised from its thread
- P11: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant; reason summarised from its thread
- P12: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P13: [Start WAL receiver eagerly](https://commitfest.postgresql.org/patch/3376/) (cf, Returned with feedback, 2022-02-07) - used in the review
- P14: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant
- P15: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P16: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P17: [Track replay lag in pg_stat_replication](https://commitfest.postgresql.org/patch/920/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P18: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - used in the review
- P19: [Transaction traceability - txid_convert_if_recent(bigint)](https://commitfest.postgresql.org/patch/748/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P20: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P21: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant
- P22: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P24: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0009-Skip-FPI-lookups-when-no-FPIs-are-pending.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 122 s; cost reported by the CLI: $1.63
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:19.981Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
