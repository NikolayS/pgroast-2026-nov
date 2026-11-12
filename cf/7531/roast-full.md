# Design pre-review: Hash index inserts serialise on the metapage tuple counter

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7531](https://commitfest.postgresql.org/patch/7531/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Hash index inserts serialise on the metapage tuple counter](https://www.postgresql.org/message-id/flat/ats7u1ISmgZaWcge%40floki), started 2026-10-23
- Messages read: 10 of 10
- Patch prose read: v4-0001-Don-t-make-hash-index-inserts-serialize-on-the-me.patch, v4-0002-Allocate-and-free-hash-overflow-pages-without-the.patch, metacount.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is stated and measured, and the one reviewer supports fixing it, so level 1 holds. The design has open points at level 2. Counts lost at backend exit add up on insert-only indexes with nothing to recount them (replication-ondisk-1). v2 brought batching back, so split timing again depends on client count and session length, and David Geier's objection has no answer (maintenance-fit-1). The list should settle these before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the 0002 bitmap locking protocol, the cached-metapage refresh rule, the redo of the optional metapage reference and the lifetime of pending-count entries could not be checked. Only 3 of 10 messages were visible, so later answers in the thread may be missed.

## The proposal, restated

**Problem.** Every hash index insert takes the metapage's exclusive content lock to bump hashm_ntuples and test the split threshold. With concurrent inserters this one buffer serializes them, so single-row insert throughput peaks at 16 clients and falls, while btree keeps scaling. In a growing index, overflow page allocation and free also lock the metapage and the bitmap page.

**Proposed design.** 0001: an insert that finds the metapage lock taken does not wait; it keeps the tuple as a pending count in a backend-local hash table keyed by index OID, flushed by a later insert that gets the lock. After 1024 inserts and contention, a backend tries only every 32nd insert or after adding an overflow page, and waits at 256 pending. 0002 searches bitmap pages under a share lock, extends by 64 overflow pages at once, and frees without the metapage.

**User-visible surface.**
- XLOG_HASH_INSERT changes: the metapage block reference is optional and may carry a tuple count increment; needs an XLOG_PAGE_MAGIC bump.
- XLOG_HASH_ADD_OVFL_PAGE gains a page count and registers the metapage only when it changes.
- pg_waldump output for hash insert records shows "ntuples +N".
- New injection point hash-insert-defer-count.
- A growing index may hold up to 63 unused overflow pages from the last batch.

**What the author claims.**
- Without concurrent inserters the count is exact and split timing matches master.
- hashm_ntuples only times splits, so a count that lags by under 256 tuples per backend only delays a split.
- On 64 vCPUs, single-row inserts reach about 686k tps at 256 clients against 50k on master, level with btree.
- Mixed, single-key, delete-heavy and high-duplicate runs give the same buckets and row counts as master, and a standby replays to the same metapage.
- Single-client INSERT ... SELECT shows no regression, and WAL per insert drops slightly.

**Thread state.** David Geier supported fixing the problem and objected to non-deterministic, concurrency-dependent split timing in the first batching design; he has not replied since v1. The author posted v1 to v4 alone, and v4 only fixes tests that failed on cfbot.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Uncounted tuples persist in hashm_ntuples with no repair on insert-only tables

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A pooler recycles backends after a few thousand inserts each, under contention. Each exit drops up to 255 pending tuples per index into the on-disk metapage permanently. On an append-only table VACUUM never recounts, so splits fall further behind, bucket chains lengthen and probes slow. pg_upgrade carries the low count forward.

Message #7 shows that a count far below reality makes _hash_expandtable() refuse splits, so chains grow. The 1.4% worst case in message #8 was for 10-insert sessions; sessions just past the 1024-insert threshold can lose up to 255 of about 1300 tuples. Message #2 notes that VACUUM recounts only when it removes tuples and sees no concurrent split or insert. The proposal offers no recount path for insert-only indexes.

Thread: [#2 (David Geier, 2026-10-26)](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#7 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me), [#8 (Greg Burd, 2026-10-28)](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki)

**Question:** How far does hashm_ntuples drift on an insert-only table when backends exit after about 1100 to 2000 contended inserts, and what corrects it?

Counter-check: held. The 1.4% figure in message #8 covers 10-insert sessions only. No run measures sessions that exit soon after 1024 inserts with up to 255 tuples pending, and message #7 confirms nothing recounts an insert-only index.

### 2. Hard-coded constants make split timing depend on session length and concurrency

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A user reports that a hash index grew unusually long overflow chains under a pooler with mid-length sessions. A developer cannot reproduce it, because whether a split fires depends on client count, on whether each backend crossed 1024 inserts, and on the every-32nd-insert phase. Nothing logs or exposes the pending counts.

In message #2 David Geier objected that batched counting makes splitting "workload dependent and by that non-deterministic", and in message #4 he asked that any alternative be checked against that problem. v1 met that goal only for uncontended inserts. v2 brought batching back after 1024 inserts, tuned on one 48-core machine (message #8), and the constants 1024, 32, 256 and 64 are fixed. No one has replied to v2 through v4 to say the objection no longer applies.

Thread: [#2 (David Geier, 2026-10-26)](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#4 (David Geier, 2026-10-26)](https://www.postgresql.org/message-id/d3704135-7f10-4457-8939-d6ad964122e5%40gmail.com), [#8 (Greg Burd, 2026-10-28)](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki)

**Question:** How would someone debugging a split-timing report find out the current pending counts, and why are 1024, 32 and 256 right on hardware other than the one test machine?

Counter-check: held. v2 brought batching back after 1024 inserts, so split timing under contention again depends on client count and session length. David Geier's objection from messages #2 and #4 has no answer for v2 to v4, and the constants come from one machine.

### 3. Pending counts keyed by index OID can flow into a new relfilenode

fixable · level 2, rubric item 14 · design, on-disk format / pg_upgrade, catalog changes · `replication-ondisk-2`

A backend holds pending counts for an index. Another session runs TRUNCATE or REINDEX, which writes a new metapage with ntuples 0. The first backend's next insert flushes its stale pending count into the new metapage, and a near-empty index then splits early and carries a wrong count until a VACUUM removes tuples.

The commit message keys the backend-local hash table by index OID. TRUNCATE, REINDEX and dropping the index with later OID reuse all keep or reuse that OID with a new relfilenode. The proposal does not say when an entry is discarded: on relcache invalidation, on relfilenode change, or on drop. The brief lists this as unclear, and no reply in the thread covers it.

Thread: [#1 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/ats7u1ISmgZaWcge%40floki), [#7 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me)

**Question:** When is a pending-count entry discarded, so that a count from an old relfilenode can never reach a new metapage?

Counter-check: held. The harm is small, since a stale flush adds fewer than 256 tuples per backend and only causes an early split. But the proposal does not say when an entry keyed by index OID is discarded, and the brief lists this as open.

### 4. Fixed 64-page overflow batch adds persistent size to small indexes

fixable · level 2, rubric item 4 · on-disk format / pg_upgrade · `replication-ondisk-3`

A schema has 1000 partitions, each with a small hash index that needs its first overflow page. Each extension adds 64 pages instead of 1, about 504 KiB per index, so about 500 MiB of unused space persists on disk and in base backups. The space is never returned.

Message #9 and the 0002 commit message extend by 64 overflow pages whenever none are free, whatever the index size. The only measurement is one large delete-heavy index, which grew by 41 pages (message #9). Small indexes, which gain nothing from contention relief, pay the same fixed cost.

Thread: [#9 (Greg Burd, 2026-11-03)](https://www.postgresql.org/message-id/aun2bbW2schM3Gpp%40floki)

**Question:** Could the batch size scale with the index size or the number of existing overflow pages, and what does 0002 add for a hash index with 10k rows?

Counter-check: held. The arithmetic is right and 0002 uses a fixed batch of 64 at any index size. The README says only the last page is written, so file systems with holes may use less disk, but no small-index measurement is shown.

### 5. No field reports of hash index users hitting this contention

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `demand-1`

A committer takes on two WAL record changes, a deferred-count scheme and a new bitmap lock protocol in the hash AM. Few production users run many concurrent single-row inserts into a hash index, so the gain reaches few people. Nobody who knows the area is engaged, so later bugs in split timing or overflow allocation wait for an owner.

All demand in the material comes from the author's own benchmarks, and message #1 says the problem was found while chasing something else. David Geier's +1 in message #2 is the only other voice, and he has not replied since v1. No -general or -bugs report, customer workload or other system's experience is cited. The 10x collapse is real on the benchmark, but it does not show how many users run that workload.

Thread: [#1 (Greg Burd, 2026-10-23)](https://www.postgresql.org/message-id/ats7u1ISmgZaWcge%40floki), [#2 (David Geier, 2026-10-26)](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#7 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me), [#8 (Greg Burd, 2026-10-28)](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki), [#9 (Greg Burd, 2026-11-03)](https://www.postgresql.org/message-id/aun2bbW2schM3Gpp%40floki)

**Question:** Which real workloads run highly concurrent inserts into hash indexes today, and are there field reports of them hitting this metapage wait?

Counter-check: held. The benchmark shows a real decline past 16 clients, but the material cites no field report or user workload. The point is sharper for 0002, which adds a new locking protocol for a 2-5% gain; this is a show-demand item.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | hashm_ntuples may stay low on disk, hashm_firstfree changes meaning, and an index may keep up to 63 unused overflow pages. |
| WAL / replication / standby | **yes** | XLOG_HASH_INSERT and XLOG_HASH_ADD_OVFL_PAGE change format, and redo applies a tuple count increment. |
| wire protocol | no |  |
| catalog changes | **yes** | see objection replication-ondisk-2: Pending counts keyed by index OID can flow into a new relfilenode |
| ABI / extension API | **yes** | hash.h and hash_xlog.h change, so external tools that decode hash WAL records must follow. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Both patches change WAL records and need an XLOG_PAGE_MAGIC bump, so they are for master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patches add a deferred-count scheme with four fixed constants, a new bitmap lock protocol and a new lock order in the hash AM. |
| performance on the hot path | **yes** | 0001 changes _hash_doinsert, which every hash index insert runs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How far does hashm_ntuples drift on an insert-only table when backends exit after about 1100 to 2000 contended inserts, and what corrects it?
2. Does split timing in v4 still depend on client count and session length, and if so, why is that acceptable given the objection in message #2?
3. When is a pending-count entry discarded, so that a count from before TRUNCATE, REINDEX, a drop with OID reuse or a subtransaction abort never reaches a metapage?
4. What does 0002 add on disk for a hash index with 10k rows, and could the 64-page batch scale with index size?
5. Which real workloads run many concurrent single-row inserts into hash indexes, and are there field reports of this metapage wait?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the hash AM's locking rules and hash WAL replay. None has spoken in the visible thread.
- **Liability:** The owner signs for two WAL record changes, a new meaning of hashm_firstfree, a new lock order and an approximate hashm_ntuples, for as long as the hash AM exists.
- **Author follow-up:** The author posted v1 to v4 within three weeks and fixed the cfbot test failures in v4, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The gain is large on the benchmark, but it reaches only users with many concurrent inserters into a hash index, and no field report is cited. The two patches change WAL records and hash AM locking, so review is costly, and the drift and split-timing questions should be settled first.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Lost pending counts accumulate in visible counts with no recount. Refuted (not specific): Duplicate of replication-ondisk-1, which states the drift with numbers. Also, on an insert-only table VACUUM does not call hashbulkdelete, so it does not copy hashm_ntuples into reltuples in this scenario.
- `maintenance-fit-2` New hash locking protocol has no reviewer who knows the area. Refuted (no failure scenario): The scenario is a hypothetical later change that ignores the README, which documents the new hashm_firstfree meaning and lock order. A thread three weeks old without a second reviewer is a committability note and shows no design flaw.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P16] covers contention on the shared dynahash HASHHDR.mutex, which is unrelated to hash index metapage locking, and it has no recorded reason. Message #1 points to the v10 metapage cache work, where inserts were noted to still touch the metapage. The thread reports that as a gap left open, not as a rejected attempt.
- standards-compat: 1 objection(s). The proposal adds no SQL syntax, GUC, catalog object or pg_dump state. The WAL format changes are master-only with an XLOG_PAGE_MAGIC bump, which is normal practice. The remaining user-facing change is the meaning of the stored tuple count.
- replication-ondisk: 3 objection(s). The WAL format changes need only an XLOG_PAGE_MAGIC bump on master. Standby replay was tested with wal_consistency_checking in message #9, so WAL itself raises no objection.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds no new privilege, role, SQL function, GUC, view or default. The pending counts are backend-local, at most one small entry per index the backend inserts into. The "ntuples +N" text appears only in pg_waldump, which already needs WAL access. Uncounted tuples left by exiting sessions can delay splits on insert-only tables, but only a user who could already bloat the index can do that, so it is a design question for other angles.
- maintenance-fit: 2 objection(s). Hot-path cost for non-users looks small, since single-client INSERT ... SELECT showed no regression in message #7. The patch set is split so that 0001 can be committed alone.
- already-exists: 0 objection(s). No existing feature, setting or extension removes the metapage lock from hash index inserts, and an extension cannot change the internals of a core access method. Switching to btree avoids the bottleneck but does not fix hash indexes, and [P16] concerns dynahash, not hash indexes.
- demand: 1 objection(s). The problem is stated with measurements. The open point is how many users it affects, which only field evidence can settle.

### Precedents retrieved

- P1: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - judged not relevant
- P2: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant; reason summarised from its thread
- P3: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant; reason summarised from its thread
- P4: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P5: [split tablecmds.c](https://commitfest.postgresql.org/patch/6269/) (cf, Withdrawn, 2026-08-24) - judged not relevant; reason summarised from its thread
- P6: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P7: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P8: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P10: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant
- P11: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P12: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P13: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - judged not relevant
- P14: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P15: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P16: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - used in the review
- P17: [Optionally record Plan IDs to track plan changes for a query](https://commitfest.postgresql.org/patch/5486/) (cf, Returned with feedback, 2025-03-24) - judged not relevant
- P18: [hashjoin - gracefully increasing NTUP_PER_BUCKET instead of batching](https://commitfest.postgresql.org/patch/28/) (cf, Rejected, 2015-02-13) - judged not relevant
- P19: [Revert changes to pg_basebackup and pg_waldump usage() code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=05b5feb60e7cf36db08eb0d34dbd21854cb9e8ce) (revert, Reverted, 2017-05-17) - judged not relevant
- P20: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant
- P21: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P22: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P23: [Split StdRdOptions into HeapOptions and ToastOptions](https://commitfest.postgresql.org/patch/2370/) (cf, Returned with feedback, 2021-10-04) - judged not relevant
- P24: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 172 s; cost reported by the CLI: $2.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:05:25.332Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
