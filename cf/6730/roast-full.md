# Design pre-review: Skip prefetch for block references that follow a FPW or WILL_INIT of the same block

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6730](https://commitfest.postgresql.org/patch/6730/) (Needs review); authors: SATYANARAYANA NARLAPURAM (satyanarayanan)
- Thread: [Skip prefetch for block references that follow a FPW or WILL_INIT of the same block](https://www.postgresql.org/message-id/flat/CAHg%2BQDf%3DeFFnOvmaPJQrXNiXaTOhhJhvO%3DDzmVUi%3DVGFRh5B0A%40mail.gmail.com), started 2026-03-24
- Messages read: 17 of 17
- Patch prose read: 0001-xlogprefetcher-record-FPW-WILL_INIT-blocks-in-the-re.patch, 0001-xlogprefetcher-record-recent-fpw.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with measurements, the thread shows demand from several independent people, and the change reuses the existing recent-blocks window in one file. Prefetch is advice only, so a skipped prefetch cannot change replay results. The open points are scope decisions for a committer: back-patching, v3 versus the hash-table variant, and the planned prefetcher rework.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 17 messages and the prose of v1 and v2 were read; v3 and the hash-table variant were not seen. The back-patch risk and the CPU cost of the added bookkeeping in replay cannot be judged without the code.

## The proposal, restated

**Problem.** During recovery, the WAL prefetcher issues posix_fadvise for a block whose earlier record in the look-ahead window carried a full page image or BKPBLOCK_WILL_INIT, so replay overwrites the page anyway. Each such call can cost a real 8 KiB read. Standbys and crash recovery with recovery_prefetch enabled pay this, and the cost is highest on IOPS-limited cloud volumes.

**Proposed design.** The patch moves the recent-blocks bookkeeping into a helper, XLogPrefetcherAddRecent(), and calls it on the FPW and WILL_INIT skip paths in XLogPrefetcherNextBlock(). The existing duplicate check then suppresses later references to the same block and counts them as skip_rep. The window size, XLOGPREFETCHER_SEQ_WINDOW_SIZE, stays unchanged.

**User-visible surface.**
- pg_stat_recovery_prefetch counters change: fewer prefetch, more skip_rep for the same WAL.

**What the author claims.**
- On NVMe, replay of 10 GB of insert-only WAL took about 31% less time, with about 90% fewer fadvise calls.
- On a disk with about 2 ms latency, the same replay was about 3.1 times faster.
- The fix may not help when a table has more than four indexes, because the window holds only a few blocks.
- A larger window or a hash table could remove more prefetches but costs CPU; the author leaves that for a separate patch.

**Thread state.** Ethan Mertz marked it Ready for Committer and Ashutosh Sharma posted v3; Bharath Rupireddy and Valeriy Degtyarev asked for a back-patch, and Bharath posted a hash-table variant for HEAD. Thomas Munro is studying the reports and an alternative idea, while others note that fadvise disables kernel read-ahead, which this patch does not address.

## 2. Prior attempts

### Revert recovery prefetching feature. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAOuzzgrn7iKnFRsB4MHp3UisEQAGgZMbk_ViTN4HV4-Ksq8zCg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAOuzzgrn7iKnFRsB4MHp3UisEQAGgZMbk_ViTN4HV4-Ksq8zCg@mail.gmail.com), [https://www.postgresql.org/message-id/flat/20210409033703.GP6592@telsasoft.com](https://www.postgresql.org/message-id/flat/20210409033703.GP6592@telsasoft.com), [https://www.postgresql.org/message-id/flat/CA+hUKGJ4VJN8ttxScUFM8dOKX0BrBiboo5uz1cq=AovOddfHpA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGJ4VJN8ttxScUFM8dOKX0BrBiboo5uz1cq=AovOddfHpA@mail.gmail.com), [https://www.postgresql.org/message-id/flat/20190418.210257.43726183.horiguchi.kyotaro@lab.ntt.co.jp](https://www.postgresql.org/message-id/flat/20190418.210257.43726183.horiguchi.kyotaro@lab.ntt.co.jp), [https://www.postgresql.org/message-id/flat/16997-ff16127f6e0d1390@postgresql.org](https://www.postgresql.org/message-id/flat/16997-ff16127f6e0d1390@postgresql.org), [https://www.postgresql.org/message-id/flat/20210413123139.GE6091@telsasoft.com](https://www.postgresql.org/message-id/flat/20210413123139.GE6091@telsasoft.com)
- **Date:** decided 2021-05-10; started 2021-04-08
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This set of commits has some bugs with known fixes, but at this late stage in the release cycle it seems best to revert and resubmit next time, along with some new automated test coverage for this whole area. Commits reverted: dc88460c: Doc: Review for "Optionally prefetch referenced data in recovery." 1d257577: Optionally prefetch referenced data in recovery. f003d9f8: Add circular WAL decoding buffer. 323cbe7c: Remove read_page callback from XLogReader. Remove the new GUC group WAL_RECOVERY recently added by a55a9847, as the corresponding section of config.sgml is now reverted.
- **Reason (quote):** "This set of commits has some bugs with known fixes, but at this late stage in the release cycle it seems best to revert and resubmit next time, along with some new automated test coverage for this whole area."
- **What is different this time:** P1 reverted the whole recovery prefetch feature before release because of late-cycle bugs and missing tests. This patch is a small change to the duplicate-detection window of the feature that went in later, and it does not change replay results.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes which blocks the recovery prefetcher sends to posix_fadvise during standby replay and crash recovery. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Two participants asked for a back-patch, and the patch does not state the risk for stable branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The patch adds recent-blocks bookkeeping on the FPW and WILL_INIT skip paths of XLogPrefetcherNextBlock(), which runs for every block reference in replay. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the patch meant for back-patching, and what is the risk in stable branches if pg_stat_recovery_prefetch counts shift from prefetch to skip_rep?
2. What does v3 gain on a table with more than four indexes, where the window of XLOGPREFETCHER_SEQ_WINDOW_SIZE blocks left most prefetches in place?
3. Should the hash-table variant for HEAD replace v3 or follow it, and what replay CPU cost does it add?
4. How does this change fit with the planned AIO or read_stream rework of the prefetcher: does it survive that rework or only cover the releases before it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Thomas Munro, who knows xlogprefetcher.c, takes part in the thread and is studying the reports and an alternative idea.
- **Liability:** The owner signs for replay performance on standbys and in crash recovery, in every branch that receives the change. The code is small, and a later prefetcher rework may replace it.
- **Author follow-up:** The author rebased the patch in message #2. The brief says another contributor posted v3, so it is not shown whether the author is still active.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is under 4 KiB in one file and adds no user-visible surface beyond shifted counter values. The thread reports a production standby problem and several independent reproductions, and the author measured about 31% less replay time on NVMe.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent rejected this idea. P8 prefetched WAL segment files, not data blocks, and P1's revert reason (late-cycle bugs, missing tests) is about the original feature, so neither applies here. The thread mentions two earlier discussions, a March 2025 report of the same behaviour (message #4) and a 2021 hash-table discussion (message #7), but gives no rejection reason for either.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, GUC, catalog object or dump state, and replay results do not change. The only visible change is that pg_stat_recovery_prefetch counts some blocks as skip_rep instead of prefetch, which still fits that counter's documented meaning as a repeated block, even in a back-patch. The open questions about back-patch risk and kernel read-ahead belong to other angles.
- replication-ondisk: 0 objection(s). The patch changes only in-memory bookkeeping in xlogprefetcher.c. WAL format, on-disk files, catalogs, the wire protocol and struct layouts all stay the same, so there is nothing to carry across pg_upgrade or between primary and standby. Prefetch is only advice: when a later reference skips prefetch, replay still reads the block normally, so no recovery invariant depends on the change. The only visible effect is that pg_stat_recovery_prefetch counts differ, and those counters are not persistent. Back-patching raises no ABI or replication-compatibility concern from this angle.
- security: 0 objection(s). The patch only changes which blocks the recovery prefetcher sends to posix_fadvise inside the startup process. It adds no privilege, role, GUC, SQL surface, view column or authentication path, so no new exposure to a non-superuser. The only visible change is counter values in pg_stat_recovery_prefetch, which already exist.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle the patch is a small, local change to one file. It adds bookkeeping only on paths that already skip a prefetch and uses the existing recent-blocks window, so it adds no API, GUC or hot-path cost for users who do not use recovery prefetch. Committability: Thomas Munro, who knows xlogprefetcher.c, is engaged in message #17 and plans an AIO rework, so v3 may be short-lived code, but it is cheap to carry. Two points could not be checked without the code: the back-patch risk in stable branches, and the absence of a regression test for the new skip path. Neither supports a level 1-2 objection on its own.
- already-exists: 0 objection(s). The only existing alternative is recovery_prefetch = off, and it does not do the same job: it also drops useful prefetches, as cases 7 and 8 in message #7 and the report in message #4 show. The patch extends the existing recent-blocks window instead of adding a second mechanism, and no extension or hook covers recovery prefetch.
- demand: 0 objection(s). The material shows real demand: a production outage report with numbers on a remote_apply standby (message #4), an independent earlier report cited there, and reproductions by several independent people (messages #6, #7, #13). The cost falls on the recovery path only, and the default recovery_prefetch = try exposes many users.

### Precedents retrieved

- P1: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - used in the review
- P2: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P3: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P4: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P5: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P6: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant; reason summarised from its thread
- P7: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P8: [Use fadvise in wal replay](https://commitfest.postgresql.org/patch/3707/) (cf, Returned with feedback, 2023-04-09) - used in the review; reason summarised from its thread
- P9: [Memory prefetching while sequentially fetching from SortTuple array, tuplestore](https://commitfest.postgresql.org/patch/305/) (cf, Returned with feedback, 2015-12-13) - judged not relevant
- P10: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P11: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P12: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant
- P13: [Track replay lag in pg_stat_replication](https://commitfest.postgresql.org/patch/920/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P14: [low wal_retrieve_retry_interval causes missed signals on Windows](https://commitfest.postgresql.org/patch/4118/) (cf, Withdrawn, 2023-02-25) - judged not relevant
- P15: [Revert "Optimize pg_visibility with read streams."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ddfc556a644404a8942e77651f75f09aa5188782) (revert, Reverted, 2024-09-04) - judged not relevant
- P16: [Revert recent ill-advised test case changes.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=82023d47de9e262730b1f9b4ea77fae201a89d0a) (revert, Reverted, 2024-04-19) - judged not relevant
- P17: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P19: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P20: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P21: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant
- P22: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P23: [Start WAL receiver eagerly](https://commitfest.postgresql.org/patch/3376/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P24: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 97 s; cost reported by the CLI: $2.05
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:01:38.537Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
