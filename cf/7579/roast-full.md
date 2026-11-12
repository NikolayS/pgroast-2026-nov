# Design pre-review: Fix ReadStream handling when heap rescan strategy changes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7579](https://commitfest.postgresql.org/patch/7579/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Fix ReadStream handling when heap rescan strategy changes](https://www.postgresql.org/message-id/flat/aumB7BHGWewR4x%2FS%40bdtpg), started 2026-11-03
- Messages read: 6 of 6
- Patch prose read: v1-0001-Fix-ReadStream-handling-when-heap-rescan-strategy.patch, v1-0002-Test-heap-rescan-strategy-changes.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic; the only one raised, maintenance-fit-1, was refuted because the material does not show a field-by-field copy in heapam.c. The proposal is a bug fix for b7b0f3f27241 with a stated defect, a reachable case in core (message #3) and no new user-visible surface. Two reviewers support a separate fix. The result is provisional because the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the read_stream.c change, its ABI effect on back branches, and the place where the statistics carry-over lives could not be checked. Stream recreation during parallel or sync scans also could not be checked.

## The proposal, restated

**Problem.** Since b7b0f3f27241, heap scans keep their ReadStream across rescans, but initscan() can add or free rs_strategy on a rescan. Enabling the strategy leaves the stream with a NULL strategy and its old queue capacity. Disabling it frees the strategy while the stream still points to it, which an extension can reach but core cannot, per the thread.

**Proposed design.** On rescan, initscan() ends and recreates the ReadStream when the scan switches between using and not using a buffer access strategy. Ordinary rescans still reset and reuse the existing stream. With instrumentation on, the new stream gets the accumulated statistics back and keeps the maximum capacity seen so far.

**User-visible surface.**
- New TAP test src/test/modules/test_misc/t/016_heap_rescan_strategy.pl, which starts its own cluster.

**What the author claims.**
- The fix uses only existing ReadStream APIs, so backpatching is easier than with the proposed read_stream_set_strategy() API.
- Against the read_stream_set_strategy() approach, the difference was below 0.15 microseconds per rescan, so performance favors neither.
- An extension can reach the dangling pointer by starting a large seqscan with allow_strat = true, then calling table_rescan_set_params() with allow_strat = false.

**Thread state.** Nazir Bilal Yavuz supports a separate fix and prefers this simpler approach for backpatching; shihao zhong agreed an extension can reach the bug and asked to say so in the commit message. Open: whether to commit the costly 0002 test, and a cosmetic condition rewrite.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes read_stream.c, and the reported dangling pointer is reached by extensions that call table_rescan_set_params(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable branches, and message #1 gives easier backpatching as a reason for this approach. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The recreate path carries instrumentation statistics and maximum capacity from the old stream to the new one. |
| performance on the hot path | **yes** | initscan() runs on every heap rescan, though the stream is recreated only when strategy use switches. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which back branches does the fix target, and does the read_stream.c change alter any exported struct or function that those branches must keep ABI-stable?
2. What changes in read_stream.c, given that message #1 says the fix uses only existing ReadStream APIs?
3. Does the statistics and maximum-capacity carry-over live inside read_stream.c, so that a new instrumentation field cannot be missed in heapam.c?
4. Is recreating the stream in initscan() safe for parallel scans and sync scans, where scan position state is shared or reported?
5. Should the 0002 TAP test, which starts its own cluster, go into the tree, or is there a cheaper way to cover the strategy switch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the ReadStream and AIO code and the heap scan setup in heapam.c would have to own it. The visible thread and the CommitFest entry do not identify any participant as a committer.
- **Liability:** The owner signs for a backpatched change to heap rescan and ReadStream lifetime in the stable branches, including any ABI effect of the read_stream.c change, for the life of those branches.
- **Author follow-up:** The author posted a patch and a test and asked for review in message #1. Only the first 3 of 6 messages are shown, so there is no further sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is a small patch to two files that corrects a defect from b7b0f3f27241. Core reaches only the stale NULL strategy case with extra buffer pollution (message #3); the freed-pointer case needs an extension.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Carrying stream statistics across recreated streams is a hand-kept copy. Refuted (no failure scenario): The scenario depends on a field-by-field copy in heapam.c, which the material does not show. The commit message says the fix reattaches the accumulated statistics, and the patch touches read_stream.c, so the carry-over may already live there.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P10] shares only the word rescan: that revert was about Gather destroying a parallel context that rescans still needed, while this patch rebuilds a ReadStream that a rescan has made stale. Message #1 points to an earlier alternative, Yuhang's read_stream_set_strategy() API. The thread reports it as reviewed with no correctness problem found, not rejected, and message #2 prefers this simpler fix for backpatching.
- standards-compat: 0 objection(s). From a standards and compatibility view this is an internal bug fix: it adds no SQL syntax, GUC, catalog object, dump state or psql behaviour, and existing queries keep their plans and results. Without the code, two points cannot be checked: whether the read_stream.c change alters any exported struct or function that back branches must keep ABI-stable, and which branches the fix targets.
- replication-ondisk: 0 objection(s). The fix changes only per-backend scan state: when the ReadStream is recreated and what strategy pointer it holds. It touches no on-disk format, WAL, catalog or wire protocol. The read_stream.c change is not described, so whether it alters a struct that extensions see in back branches cannot be checked without the code.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It fixes a dangling pointer that only an extension calling table_rescan_set_params() can reach, so no SQL-level user gains a new path. It adds no new object, role, setting, view or default. Stream recreation runs only on a strategy switch, so an unprivileged user gets no new resource-use lever.
- maintenance-fit: 1 objection(s). The fix itself fits core and has no hot-path cost. Ordinary rescans reuse the stream, and message #5 measured a difference of less than 0.15 microseconds per rescan. The only concern from this angle is the instrumentation carry-over.
- already-exists: 0 objection(s). This is a bug fix for a ReadStream that keeps a stale or freed strategy across heap rescans, and no existing feature, extension or setting avoids it. The only alternative, read_stream_set_strategy() from another thread, is a competing unmerged fix, and this patch builds on existing ReadStream APIs without adding a second mechanism.
- demand: 0 objection(s). This is a bug fix to b7b0f3f27241, so it needs a reachable defect, not user demand. Core can reach the stale NULL strategy and its extra buffer pollution (message #3), and an extension can reach the freed strategy pointer (messages #4 and #6). Two reviewers agree a separate fix is worth doing, and the fix adds no user-visible surface. Whether to commit the costly 0002 test is a test-cost question outside the demand angle.

### Precedents retrieved

- P1: [Eagerly evict bulkwrite strategy ring](https://commitfest.postgresql.org/patch/5919/) (cf, Withdrawn, 2026-02-19) - judged not relevant; reason summarised from its thread
- P2: [Implement TODOs for index operator strategy number validation](https://commitfest.postgresql.org/patch/7152/) (cf, Withdrawn, 2026-09-09) - judged not relevant; reason summarised from its thread
- P3: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P4: [Revert "Optimize pg_visibility with read streams."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ddfc556a644404a8942e77651f75f09aa5188782) (revert, Reverted, 2024-09-04) - judged not relevant
- P5: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P6: [Revert "Add eager and lazy freezing strategies to VACUUM."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c6b4972664491de408670354fdbccc7c83a1c89) (revert, Reverted, 2023-01-26) - judged not relevant
- P7: [Revert the default toast compression strategy to the former behavior where there's no limit on the size of datum we'll try to compress.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=229bffedbc52f12aadb95ca2ee24d5bef36b7c7d) (revert, Reverted, 2009-01-06) - judged not relevant
- P8: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [Memory prefetching while sequentially fetching from SortTuple array, tuplestore](https://commitfest.postgresql.org/patch/305/) (cf, Returned with feedback, 2015-12-13) - judged not relevant
- P10: [Don't shut down Gather\[Merge\] early under Limit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=080313f8296fb0bcc74bd70fc8e15cd64f45945e) (revert, Partially reverted, 2019-11-26) - used in the review
- P11: [Reorder tablespaces for streaming basebackups](https://commitfest.postgresql.org/patch/1035/) (cf, Rejected, 2017-03-31) - judged not relevant; reason summarised from its thread
- P12: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P13: [Revert "Add USER SET parameter values for pg_db_role_setting"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b9a7a822723aebb16cbe7e5fb874e5124745b07e) (revert, Reverted, 2023-05-17) - judged not relevant
- P14: [Improve documentation for pg_upgrade, standbys and rsync](https://commitfest.postgresql.org/patch/3125/) (cf, Withdrawn, 2022-04-08) - judged not relevant; reason summarised from its thread
- P15: [Set default transactions to read-only at servers start in pg_upgrade](https://commitfest.postgresql.org/patch/2970/) (cf, Withdrawn, 2021-12-01) - judged not relevant; reason summarised from its thread
- P16: [allow specifying action when standby encounters incompatible parameter settings](https://commitfest.postgresql.org/patch/3619/) (cf, Withdrawn, 2022-07-25) - judged not relevant
- P17: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P18: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P19: [Detect supported SET parameters when pg_restore is run](https://commitfest.postgresql.org/patch/803/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P20: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P21: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant
- P22: [Optimization for lazy_scan_heap](https://commitfest.postgresql.org/patch/699/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P23: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - judged not relevant
- P24: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 112 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:41.760Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
