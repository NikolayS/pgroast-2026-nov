# Design pre-review: Hash index inserts serialise on the metapage tuple counter

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7531](https://commitfest.postgresql.org/patch/7531/); [thread](https://www.postgresql.org/message-id/flat/ats7u1ISmgZaWcge%40floki), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is stated and measured, and the one reviewer supports fixing it, so level 1 holds. The design has open points at level 2. Counts lost at backend exit add up on insert-only indexes with nothing to recount them (replication-ondisk-1). v2 brought batching back, so split timing again depends on client count and session length, and David Geier's objection has no answer (maintenance-fit-1). The list should settle these before code review.

Not checked: Without the code, the 0002 bitmap locking protocol, the cached-metapage refresh rule, the redo of the optional metapage reference and the lifetime of pending-count entries could not be checked. Only 3 of 10 messages were visible, so later answers in the thread may be missed.

## Prior attempts

None found before 2026-10-23.

## Objections a reviewer is likely to raise

### 1. Uncounted tuples persist in hashm_ntuples with no repair on insert-only tables

A pooler recycles backends after a few thousand inserts each, under contention. Each exit drops up to 255 pending tuples per index into the on-disk metapage permanently. On an append-only table VACUUM never recounts, so splits fall further behind, bucket chains lengthen and probes slow. pg_upgrade carries the low count forward.

Message #7 shows that a count far below reality makes _hash_expandtable() refuse splits, so chains grow. The 1.4% worst case in message #8 was for 10-insert sessions; sessions just past the 1024-insert threshold can lose up to 255 of about 1300 tuples. Message #2 notes that VACUUM recounts only when it removes tuples and sees no concurrent split or insert. The proposal offers no recount path for insert-only indexes.

**Question:** How far does hashm_ntuples drift on an insert-only table when backends exit after about 1100 to 2000 contended inserts, and what corrects it?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#2](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#7](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me), [#8](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki); `replication-ondisk-1`</sub>

### 2. Hard-coded constants make split timing depend on session length and concurrency

A user reports that a hash index grew unusually long overflow chains under a pooler with mid-length sessions. A developer cannot reproduce it, because whether a split fires depends on client count, on whether each backend crossed 1024 inserts, and on the every-32nd-insert phase. Nothing logs or exposes the pending counts.

In message #2 David Geier objected that batched counting makes splitting "workload dependent and by that non-deterministic", and in message #4 he asked that any alternative be checked against that problem. v1 met that goal only for uncontended inserts. v2 brought batching back after 1024 inserts, tuned on one 48-core machine (message #8), and the constants 1024, 32, 256 and 64 are fixed. No one has replied to v2 through v4 to say the objection no longer applies.

**Question:** How would someone debugging a split-timing report find out the current pending counts, and why are 1024, 32 and 256 right on hardware other than the one test machine?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#4](https://www.postgresql.org/message-id/d3704135-7f10-4457-8939-d6ad964122e5%40gmail.com), [#8](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki); `maintenance-fit-1`</sub>

### 3. Pending counts keyed by index OID can flow into a new relfilenode

A backend holds pending counts for an index. Another session runs TRUNCATE or REINDEX, which writes a new metapage with ntuples 0. The first backend's next insert flushes its stale pending count into the new metapage, and a near-empty index then splits early and carries a wrong count until a VACUUM removes tuples.

The commit message keys the backend-local hash table by index OID. TRUNCATE, REINDEX and dropping the index with later OID reuse all keep or reuse that OID with a new relfilenode. The proposal does not say when an entry is discarded: on relcache invalidation, on relfilenode change, or on drop. The brief lists this as unclear, and no reply in the thread covers it.

**Question:** When is a pending-count entry discarded, so that a count from an old relfilenode can never reach a new metapage?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/ats7u1ISmgZaWcge%40floki), [#7](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me); `replication-ondisk-2`</sub>

### 4. Fixed 64-page overflow batch adds persistent size to small indexes

A schema has 1000 partitions, each with a small hash index that needs its first overflow page. Each extension adds 64 pages instead of 1, about 504 KiB per index, so about 500 MiB of unused space persists on disk and in base backups. The space is never returned.

Message #9 and the 0002 commit message extend by 64 overflow pages whenever none are free, whatever the index size. The only measurement is one large delete-heavy index, which grew by 41 pages (message #9). Small indexes, which gain nothing from contention relief, pay the same fixed cost.

**Question:** Could the batch size scale with the index size or the number of existing overflow pages, and what does 0002 add for a hash index with 10k rows?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#9](https://www.postgresql.org/message-id/aun2bbW2schM3Gpp%40floki); `replication-ondisk-3`</sub>

### 5. No field reports of hash index users hitting this contention

A committer takes on two WAL record changes, a deferred-count scheme and a new bitmap lock protocol in the hash AM. Few production users run many concurrent single-row inserts into a hash index, so the gain reaches few people. Nobody who knows the area is engaged, so later bugs in split timing or overflow allocation wait for an owner.

All demand in the material comes from the author's own benchmarks, and message #1 says the problem was found while chasing something else. David Geier's +1 in message #2 is the only other voice, and he has not replied since v1. No -general or -bugs report, customer workload or other system's experience is cited. The 10x collapse is real on the benchmark, but it does not show how many users run that workload.

**Question:** Which real workloads run highly concurrent inserts into hash indexes today, and are there field reports of them hitting this metapage wait?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/ats7u1ISmgZaWcge%40floki), [#2](https://www.postgresql.org/message-id/1ea04014-a4e3-42d1-98aa-dd1c415fd750%40gmail.com), [#7](https://www.postgresql.org/message-id/K0iPnpAe16admdky-MM68UFoKLILFapBOyNYWrpqGerohZ970iL4pDVZg4SNPZA9WvzzR2xIC7DsfvzoBiU4XjqGbZFbMmitOJFHpXBO-xA%3D%40burd.me), [#8](https://www.postgresql.org/message-id/auHfGQ-MN1Oij_dc%40floki), [#9](https://www.postgresql.org/message-id/aun2bbW2schM3Gpp%40floki); `demand-1`</sub>

## Questions for the author

1. How far does hashm_ntuples drift on an insert-only table when backends exit after about 1100 to 2000 contended inserts, and what corrects it?
2. Does split timing in v4 still depend on client count and session length, and if so, why is that acceptable given the objection in message #2?
3. When is a pending-count entry discarded, so that a count from before TRUNCATE, REINDEX, a drop with OID reuse or a subtransaction abort never reaches a metapage?
4. What does 0002 add on disk for a hash index with 10k rows, and could the 64-page batch scale with index size?
5. Which real workloads run many concurrent single-row inserts into hash indexes, and are there field reports of this metapage wait?

## Touch points

- **on-disk format / pg_upgrade:** hashm_ntuples may stay low on disk, hashm_firstfree changes meaning, and an index may keep up to 63 unused overflow pages.
- **WAL / replication / standby:** XLOG_HASH_INSERT and XLOG_HASH_ADD_OVFL_PAGE change format, and redo applies a tuple count increment.
- **catalog changes:** see objection replication-ondisk-2: Pending counts keyed by index OID can flow into a new relfilenode
- **ABI / extension API:** hash.h and hash_xlog.h change, so external tools that decode hash WAL records must follow.
- **backpatchability:** Both patches change WAL records and need an XLOG_PAGE_MAGIC bump, so they are for master only.
- **maintenance burden ("who maintains this in 10 years"):** The patches add a deferred-count scheme with four fixed constants, a new bitmap lock protocol and a new lock order in the hash AM.
- **performance on the hot path:** 0001 changes _hash_doinsert, which every hash index insert runs.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the hash AM's locking rules and hash WAL replay. None has spoken in the visible thread.
- Priority: fix design first; audience niche; review cost high. The gain is large on the benchmark, but it reaches only users with many concurrent inserters into a hash index, and no field report is cited. The two patches change WAL records and hash AM locking, so review is costly, and the drift and split-timing questions should be settled first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
