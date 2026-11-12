# Design pre-review: Index Prefetching

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4351](https://commitfest.postgresql.org/patch/4351/) (Needs review); authors: Peter Geoghegan (pgeoghegan), Tomas Vondra (fuzzycz)
- Thread: [index prefetching](https://www.postgresql.org/message-id/flat/cf85f46f-b02f-05b2-5248-5000b894ebab%40enterprisedb.com), started 2023-06-08
- Messages read: 515 of 515
- Patch prose read: v20240124-0001-Prefetch-heap-pages-during-index-scans.patch, v20240124-0002-fixup.patch, index-prefetch-poc.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the demand, and committers who know the area wrote the series. Four fixable design points remain at level 2. They are the GiST hot standby interlock (replication-ondisk-1), the crash path for table AMs without the batch callbacks (maintenance-fit-1), the lost SP-GiST index-only scans (standards-compat-1) and the changed index tuple counter (standards-compat-2). None of them requires a different proposal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v38 code was not read. Whether RLS and leakproof quals are applied in the same order after batching, and how the full series performs across all index AMs and io_method settings, could not be checked.

## The proposal, restated

**Problem.** Plain index scans and index-only scans read heap pages one at a time and synchronously, so they cannot use the queue depth of modern storage. Bitmap heap scans already prefetch, but they do not fit queries that need index order, KNN searches on GiST/SP-GiST, or plans where the planner picks an index scan by mistake.

**Proposed design.** A new amgetbatch interface replaces amgettuple. Index AMs return a batch of matching items per leaf page into a ring buffer owned by the index scan, and heapam feeds future heap blocks to a read stream. nbtree, hash, GiST and SP-GiST adopt amgetbatch, and the series adds read stream changes such as request deduplication and suppression of posix_fadvise under io_method=sync.

**User-visible surface.**
- GUC debug_disable_indexscan_prefetch, a developer option that turns off index scan prefetching (earlier named enable_indexscan_prefetch).
- EXPLAIN (ANALYZE, IO) shows read stream instrumentation for plain index scans.
- GiST and SP-GiST no longer allow index-only scans with ordering operators such as <->, and SP-GiST disables index-only scans for long-value opclasses.
- Index AM API: new amgetbatch, amkillitemsbatch, amgettransform callbacks and an amcanmarkpos field; nbtree sets amgettuple to NULL.
- Table AM API: new required index_scan_batch_init, index_scan_markpos and index_scan_restrpos callbacks, plus tableam_indexscan.h and batchscan.h helpers.

**What the author claims.**
- Index scans with prefetching run about as fast as bitmap scans; the first message reports 16 s down to about 1.8 s for 100000 rows.
- Range scans on nbtree, GiST and SP-GiST can be 20 to 25 times faster.
- The start heuristic, which creates a read stream on the fourth heap block switch, removes the regressions with LIMIT and nestloop joins in the author's microbenchmark suite.
- Read stream deduplication turns a 1.15x slowdown into 0.58x of the master runtime on Tomas Vondra's adversarial test.
- pgstat_count_index_tuples now counts once per batch, so ORDER BY ... LIMIT 1 queries may report many more index tuples.

**Thread state.** The patch missed Postgres 19, and the slot-based table AM index scan interface was committed in September 2026; v38 carries the remaining patches toward Postgres 20. Open review points are Greg Burd's requests for out-of-core table AMs and Yuhang Qiu's GiST standby cleanup-lock concern; Bilal Kahraman asked whether the SP-GiST INCLUDE fix should be back-patched.

## 2. Prior attempts

No precedent found (corpus searched for items before 2023-06-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. GiST batches on a hot standby can return deleted tuples

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

On a hot standby, a cursor runs an index-only scan on a GiST index and holds a batch copied from the root leaf. The primary splits the root, then vacuums the heap and sets all-visible bits. Replay applies this while the cursor still holds the batch, so the cursor resumes, skips heap checks and returns rows already deleted on the primary.

Message #502 reports that the GiST cleanup-lock interlock is enforced only when VACUUM deletes leaf tuples, so the standby has no record that conflicts with the pinned root. Message #507 answers that nbtree has the same bug on master, which does not settle the point. Moving GiST to amgetbatch makes it hold copied items across pins for longer, the same pattern that exposes the bug. The thread summary lists this as an open point.

Thread: [#502 (Yuhang Qiu, 2026-09-23)](https://www.postgresql.org/message-id/ECCA330B-75BB-43BF-96E9-9049BF213019%40gmail.com), [#507 (Peter Geoghegan, 2026-10-10)](https://www.postgresql.org/message-id/CAH2-WznVcopv-p42CqmJ3FzP3EXw41nWFCeEKi1jRo3csWRoZA%40mail.gmail.com)

**Question:** Which WAL record makes standby replay wait for a GiST scan that still holds a batch from the old root, or why does amgetbatch not make this window wider?

Counter-check: held. Message #507 answers the report in #502 with a question, and the thread shows no reply. The claim that amgetbatch holds copied items for longer has no support in the material and should be dropped; the missing standby interlock stands.

### 2. Out-of-core table AMs lose btree index scans without the batch callbacks

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An extension table AM that works on Postgres 19 is loaded on Postgres 20. An assert build fails the new GetTableAmRoutine() assertion at load. A release build calls a NULL amgettuple on the first btree scan and crashes.

The brief lists index_scan_batch_init, index_scan_markpos and index_scan_restrpos as new required callbacks, and nbtree sets amgettuple to NULL. Message #515 points out this break, and the thread has no answer yet. The proposal does not say how tableam.sgml documents it, and it has no generic fallback that simple table AMs could use. Each table AM author would then copy the ring-buffer logic, which is the duplication Andres warned about in message #484.

Thread: [#515 (Greg Burd, 2026-10-29)](https://www.postgresql.org/message-id/auOon54DuG2A6ll4%40floki), [#484 (Andres Freund, 2026-04-05)](https://www.postgresql.org/message-id/6sphk3ycctmbihlrykts7uj6mjakop6wrq2dhe3vnlmrnldz2f%40uuwmkd6jjrxa)

**Question:** Will a generic batch implementation in tableam_indexscan.c let an existing table AM keep working, and where is the break documented?

Counter-check: held. Message #515 reports that a table AM without the batch callbacks calls a NULL amgettuple in a release build, and the thread has no reply. The helpers added in v27 reduce the duplication concern, but there is no generic fallback or clean error.

### 3. SP-GiST long-value opclasses lose index-only scans after upgrade

fixable · level 2, rubric item 6 · planner behaviour change, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A site uses an SP-GiST text radix index for index-only prefix lookups on a mostly all-visible table. After upgrade the planner can no longer pick an index-only scan, so every match fetches a heap page. The site finds out when latency rises or EXPLAIN stops showing Index Only Scan, with no warning at upgrade.

Message #491 states this is "a legitimate limitation in the amgetbatch design itself", with no existing bug behind it, and proposes to accept it so SP-GiST can prefetch. That withdraws a working plan type from existing indexes in exchange for a gain on another plan type. The GiST ordered index-only scan removal in message #489 is a bug fix and differs. The SP-GiST loss has no stated user notice: Tomas's documentation patch in message #509 covers only nearest-neighbor ordering.

Thread: [#491 (Peter Geoghegan, 2026-06-22)](https://www.postgresql.org/message-id/CAH2-WzkZTkDuyVFszLwPJesF9pS5E8m0UA%2B344bx-B-zfA5kaw%40mail.gmail.com), [#489 (Peter Geoghegan, 2026-06-07)](https://www.postgresql.org/message-id/CAH2-WzmAKApAYttMQfhh-usweXvRYpHcku-OKHYgDvb%3DwLSWHg%40mail.gmail.com), [#509 (Tomas Vondra, 2026-10-12)](https://www.postgresql.org/message-id/fcf9030d-150c-4ce8-89f5-83c8e5f4355f%40vondra.me)

**Question:** Which users run index-only scans on SP-GiST long-value opclasses today, and why is keeping amgettuple for those opclasses not viable?

Counter-check: held. Message #491 says the SP-GiST long-value restriction comes from the amgetbatch design and has no bug fix behind it. The thread shows no measurement of the loss and no planned user notice for this case.

### 4. pg_stat index tuple counters change meaning without user notice

fixable · level 2, rubric item 12 · design, planner behaviour change · `standards-compat-2`

A monitoring tool alerts on the idx_tup_read to idx_tup_fetch ratio in pg_stat_user_indexes. After upgrade, ORDER BY ... LIMIT 1 queries count a whole leaf page of index tuples per call, so the ratio jumps and the alerts fire. The DBA cannot tell a counting change from index bloat.

Message #492 moves pgstat_count_index_tuples to once per batch for a micro-optimization, and message #494 says the change could be "several hundred times more" and "isn't a tiny behavioral change". The argument for it is consistency with bitmap scans, but users have read this counter per plain index scan for many releases. The proposal does not say where monitoring.sgml or the release notes would describe the new meaning.

Thread: [#492 (Peter Geoghegan, 2026-07-07)](https://www.postgresql.org/message-id/CAH2-WzmsebafEMoBp-sWq%2BDp75SKV6j%2BRWerKSJNfJkWV592tg%40mail.gmail.com), [#493 (Tomas Vondra, 2026-07-08)](https://www.postgresql.org/message-id/0352479d-6a59-4e80-8ad1-12053a4ad148%40vondra.me), [#494 (Peter Geoghegan, 2026-07-11)](https://www.postgresql.org/message-id/CAH2-Wz%3DDhG5%3Dsw9P1A5%3DbnXxCz4UJaWgH5WQA-3pEckXRucZdQ%40mail.gmail.com)

**Question:** Could the counter keep its per-tuple meaning at about the same cost, and if not, where is the change documented for monitoring users?

Counter-check: held. Message #494 confirms the counter can rise several hundred times for `ORDER BY ... LIMIT 1` queries. The thread weighed the change and Tomas accepted it in #493, but the visible material names no user-facing note, so the fixable part stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | GiST scans on a hot standby hold batches while replay sets all-visible bits (messages #502, #507). |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | New amgetbatch, amkillitemsbatch and amgettransform index AM callbacks, and three new required table AM callbacks. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: SP-GiST long-value opclasses lose index-only scans after upgrade |
| new GUC | **yes** | The series adds the developer option debug_disable_indexscan_prefetch. |
| planner behaviour change | **yes** | The planner can no longer pick index-only scans for GiST and SP-GiST ordering operators or SP-GiST long-value opclasses. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The thread asks whether the SP-GiST INCLUDE fix should be back-patched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Four index AMs and every table AM must follow the new batch interface and the read stream changes. |
| performance on the hot path | **yes** | Every plain index scan and index-only scan goes through the batch path; a start heuristic guards LIMIT and nestloop cases. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which WAL record makes standby replay wait for a GiST scan that still holds a batch from the old root page?
2. Will a table AM without the batch callbacks get a generic batch implementation or a clean load-time error, and where does tableam.sgml document the change?
3. Why can SP-GiST long-value opclasses not keep amgettuple, so that their index-only scans stay available?
4. Can pgstat_count_index_tuples keep its per-tuple meaning at about the same cost, and if not, where will monitoring users read about the change?
5. What are the benchmark results for the full v38 series across nbtree, hash, GiST and SP-GiST with each io_method setting?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The CommitFest entry lists Peter Geoghegan and Tomas Vondra as authors. The owner needs to know nbtree, the index AM and table AM APIs, and the read stream code.
- **Liability:** The owner signs for a new index AM and table AM batch interface used by every plain index scan, for its hot standby behaviour, and for read stream changes. This lasts as long as the interface exists.
- **Author follow-up:** Strong sign that the authors stay: the thread runs from 2023 to v38 in 2026 across 17 CommitFests, with regular rebases.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** most users
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Index scans that read uncached heap pages are common, and message #1 reports about 16 s down to 1.8 s for 100000 rows. The series changes the index AM and table AM APIs and four index AMs, so review and upkeep are costly, but qualified committers are engaged.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Out-of-core table AMs stop working with btree indexes. Refuted (wrong facts): The slot interface was committed in September 2026 for Postgres 20, so both table AM changes land in one release and no extension was built against a Postgres 19 slot interface. The rest is a duplicate of maintenance-fit-1.
- `maintenance-fit-2` Batch interface leaves no room for per-item recheck or other verdicts. Refuted (no failure scenario): The scenario is a later index AM API change, and Postgres routinely changes AM APIs between major versions. No in-core AM needs these answers, and message #515 itself says the extra verdicts need not be in the first commit.
- `maintenance-fit-3` Some GiST and SP-GiST index-only scans are removed permanently. Refuted (not specific): Duplicate of standards-compat-1 for the SP-GiST long-value case. The GiST ordering case comes from a bug fix, and message #510 agrees to document it.
- `demand-1` SP-GiST text index-only scans removed to gain prefetching. Refuted (not specific): Duplicate of standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9], the earlier asynchronous IO patch set, was returned because it went inactive and its 39 patches were too large to review; this series is split into steps and parts are already committed. Messages #3 and #10 report that 2008-era prefetching work was limited to bitmap heap scans because index-scan block lists were hard to get and OS async IO was brittle; neither was a rejection of the idea.
- standards-compat: 2 objection(s). The GUC is a developer option that does not change query results, and the removal of GiST ordered index-only scans fixes a real bug and has a documentation patch, so neither is raised. The break for out-of-core table AMs belongs to the extension API angle.
- replication-ondisk: 2 objection(s). The proposal changes no on-disk format, catalog or wire protocol. The concerns from this angle are the hot standby cleanup-lock interlock and the break in the extension API.
- security: 0 objection(s). The proposal changes how index scans fetch heap pages. It adds no new privilege, role, SQL object or exposure, so ordinary users can do and see nothing they could not before. Batch memory is capped at 128 batches and the dedup cache at 32 blocks, and the I/O depth follows effective_io_concurrency, so an unprivileged user gets no unbounded memory or worker use. The SSI predicate-lock gap (message #500) and the SP-GiST INCLUDE crash (messages #505, #508) were fixed in the series. The GiST standby cleanup-lock issue (messages #502, #507) is a correctness concern for another angle. Without the code, it is not checked that RLS and leakproof quals are still applied after batching in the same order as on master.
- maintenance-fit: 3 objection(s). Hot-path cost and ownership look addressed. Several committers who know the area (Geoghegan, Freund, Vondra, Munro) are engaged, and regressions were measured in messages #499, #503 and #508. The nbtree VACUUM bookkeeping invariant does not apply, since the series does not change page deletion.
- already-exists: 0 objection(s). The only existing alternative is the bitmap heap scan, which already prefetches. It cannot return rows in index order or run KNN searches on GiST/SP-GiST, so it cannot do these jobs at all (message #1). No contrib module or extension adds heap prefetching inside plain index scans, and pg_prewarm loads whole relations without scan look-ahead. The read stream work extends the existing read stream mechanism and adds no second one.
- demand: 1 objection(s). Demand for the core feature is well shown. Several independent users and committers asked for it or measured it, an outside workload is described in message #10, and results include about a 10x gain on master workloads and field-style latency tests in message #504. Only the SP-GiST trade-off is questioned.

### Precedents retrieved

- P1: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P2: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P3: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant; reason summarised from its thread
- P4: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P5: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P6: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P7: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P8: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant; reason summarised from its thread
- P9: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P10: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant
- P11: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P12: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant
- P14: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P15: [Simplifying the interface of UpdateMinRecoveryPoint](https://commitfest.postgresql.org/patch/673/) (cf, Rejected, 2016-08-26) - judged not relevant
- P16: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P17: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P18: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant
- P19: [Optimization for lazy_scan_heap](https://commitfest.postgresql.org/patch/699/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P20: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - judged not relevant
- P21: [Scan key push down to heap](https://commitfest.postgresql.org/patch/850/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P22: [Support ALTER INDEX ... WHERE ... clause](https://commitfest.postgresql.org/patch/502/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P23: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - judged not relevant
- P24: [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (cf, Returned with feedback, 2017-03-07) - judged not relevant

### Warnings

- 470 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-fable-5-2, synthesis claude-fable-5-2; no tools given to any of them
- LLM calls: 16; wall time: 159 s; cost reported by the CLI: $8.75
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 88 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:54:20.472Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
