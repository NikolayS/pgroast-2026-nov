# Design pre-review: Caching a partition index's parent OID in the relcache?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7030](https://commitfest.postgresql.org/patch/7030/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [Caching a partition index's parent OID in the relcache?](https://www.postgresql.org/message-id/flat/CAJTYsWUV3Tz5A161y2EcCMqnmCYqKNm6Jt%3DrXCx9bEOJ-hqL%3DA%40mail.gmail.com), started 2026-07-17
- Messages read: 7 of 7
- Patch prose read: v2-0001-Cache-partition-index-ancestors-in-relcache.patch, v1-0001-Cache-partition-index-parent-in-relcache.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, measured, and already flagged by an XXX comment in the code. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about where the subtree invalidation rule lives and what enforces it. A committer who knows relcache invalidation has to judge that rule, together with the unanswered locking question.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether the cache is reset on every relcache path, whether the NoLock subtree scan is safe, or whether index_concurrently_swap() invalidates descendants. Only 3 of 7 thread messages were shown.

## The proposal, restated

**Problem.** For INSERT ... ON CONFLICT on partitioned tables, ExecInitPartitionInfo() calls get_partition_ancestors() once per leaf-partition index, and each call scans pg_inherits once per hierarchy level. The cost grows with the number of indexes per leaf and with partitioning depth, and it hits workloads doing repeated upserts. An existing XXX comment in that loop already flags it as slow.

**Proposed design.** v2 adds get_partition_index_ancestors(), which lazily caches the full ancestor list on the index's relcache entry as a single OID or an array in CacheMemoryContext. The cache resets on full relcache rebuild and in RelationReloadIndexInfo(). Because pg_inherits changes send no relcache invalidation, IndexSetParentIndex() now invalidates the whole descendant-index subtree via find_all_inheritors().

**User-visible surface.**
- C API: new function get_partition_index_ancestors(Relation indexRel) in catalog/partition.h.
- C API: new RelationData fields rd_partancestorcount and rd_partancestors in utils/rel.h.

**What the author claims.**
- v2 gives +16% to +31% TPS on a 64-leaf table with 4 to 15 indexes per leaf (pgbench, 8 clients).
- With 8 indexes, patched TPS stays about flat from 1 to 4 levels, while master drops from 12179 to 8442.
- In perf, get_partition_ancestors falls from about 85% of time (children) to under 2%.
- ATTACH plus DETACH with 256 leaf subpartitions goes from about 10 ms to 31 ms; the author calls this acceptable because re-parenting is rare.
- A cold first statement still scans pg_inherits; only repeated upserts benefit (stated for v1).

**Thread state.** Álvaro Herrera proposed caching the full ancestor list and supported the subtree invalidation in IndexSetParentIndex(); v2 follows that. Mihail Nikalayeu linked a related thread without reviewing; the author's questions on invalidation placement and NoLock in find_all_inheritors() have no answer.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Cache correctness depends on one function remembering to invalidate descendants

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-1`

A later patch adds a new path that changes an intermediate partitioned index's pg_inherits link without going through IndexSetParentIndex(). Leaf indexes keep stale ancestor lists. Users find out when upserts fail with "invalid arbiter index list" or resolve the wrong arbiter, with no assert or test failure to catch it earlier.

Message #3 shows that pg_inherits changes send no relcache invalidation. So v2's correctness rests on a new rule: whoever re-parents an index must invalidate the whole descendant subtree, and that rule lives only inside IndexSetParentIndex(). The proposal does not say how a future writer of pg_inherits for indexes would learn this rule, or what would fail if they missed it. The single regression test covers only the ATTACH/DETACH path, and the question in message #5 about other paths has no answer.

Thread: [#3 (Ayush Tiwari, 2026-07-17)](https://www.postgresql.org/message-id/CAJTYsWXquWQR2cY9jjfSjG7nFHhTD_GW1hFO6xnQSXBtfNk3tA%40mail.gmail.com), [#5 (Ayush Tiwari, 2026-07-17)](https://www.postgresql.org/message-id/CAJTYsWWEDMpFETWC1AE%3Dr%2Bos__R9FbK5ELFJBKEKhT%2BuuWtjEw%40mail.gmail.com), [#7 (Ayush Tiwari, 2026-07-21)](https://www.postgresql.org/message-id/CAJTYsWX-9HMD3T43txYdCubXk0FaBDMVygU3c6C3zDMqv0xDOw%40mail.gmail.com)

**Question:** Which code paths can change a partitioned index's pg_inherits link today, and what would make a future path that skips the subtree invalidation fail loudly?

Counter-check: held. Álvaro Herrera supported subtree invalidation in IndexSetParentIndex(), but the thread does not list the other paths that write pg_inherits for indexes. The v1 commit message names index_concurrently_swap() as one such path, and the question in message #5 has no answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | No catalog schema change, but the cache depends on pg_inherits rows, whose changes send no relcache invalidation. |
| ABI / extension API | **yes** | Adds rd_partancestorcount and rd_partancestors to RelationData and get_partition_index_ancestors() to catalog/partition.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future path that re-parents a partitioned index must invalidate the descendant-index subtree, and only IndexSetParentIndex() does so. |
| performance on the hot path | **yes** | Changes the arbiter-mapping loop in ExecInitPartitionInfo(); ATTACH plus DETACH with 256 leaf subpartitions slows from about 10 ms to 31 ms. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which code paths can change a partitioned index's pg_inherits link today, and what would make a future path that skips the subtree invalidation fail loudly?
2. Is NoLock in find_all_inheritors() safe on every path that calls IndexSetParentIndex(), including paths other than ATTACH and DETACH?
3. Does REINDEX CONCURRENTLY, which moves the pg_inherits link in index_concurrently_swap(), still invalidate the cached ancestor lists of descendant indexes in v2?
4. How much memory do the cached ancestor arrays use in a backend that opens many partition indexes in a hierarchy of three or more levels?
5. Can other callers of get_partition_ancestors() use the new cache, or does get_partition_index_ancestors() serve only the arbiter loop?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera proposed the full-list design in message #2 and, per the brief, supported the subtree invalidation. The owner must know relcache invalidation and partitioned-index DDL.
- **Liability:** The owner signs for the rule that every index re-parent invalidates the descendant-index subtree. A missed path gives a wrong arbiter or "invalid arbiter index list" errors, for as long as the cache exists.
- **Author follow-up:** The author posted v2 after the review in message #2, measured the ATTACH and DETACH cost, and moved the entry to the next CommitFest on 2026-10-01.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch gives 16% to 31% more TPS for repeated upserts on partitioned tables with several indexes per leaf, and a committer shaped the design. The patch is small (about 17 KB) but changes relcache invalidation, and the code was not read here.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at the same or a similar idea. Message #2 points to an earlier commit that added the arbiter-mapping code without measuring its cost, and message #6 links a related thread whose content is not shown, so no earlier rejection reason can be checked or applied.
- standards-compat: 0 objection(s). The patch adds no SQL syntax, GUC, catalog object, dump state or psql change, and query results stay the same. Its only visible effects are the slower ATTACH/DETACH on deep hierarchies and the new RelationData fields, which belong to the performance and extension API angles. It targets master only, so backpatching does not apply.
- replication-ondisk: 0 objection(s). The patch adds only backend-local relcache fields plus extra relcache invalidations from IndexSetParentIndex(). Those invalidations travel in the commit record as usual, so hot standby backends receive them. There is no on-disk, WAL-format, catalog, pg_upgrade or wire change, and the RelationData change targets master only. Whether the invalidation is correct on every re-parent path (such as DETACH CONCURRENTLY) is a design question for another angle.
- security: 0 objection(s). The proposal adds an internal relcache cache and extra invalidations, with no new privilege check, SQL surface or exposed data. The added invalidation cost lands only on ATTACH and DETACH, which need table ownership, and the cached arrays grow with the number of index relcache entries a backend opens. A stale cache would be a correctness problem, which other angles cover.
- maintenance-fit: 1 objection(s). Other maintenance points look settled. A committer who knows the area (message #2, message #4) proposed the design. Non-users gain on the hot path, and the ATTACH/DETACH cost was measured and accepted in the thread.
- already-exists: 0 objection(s). No existing feature, contrib module or extension caches a partition index's ancestor list; the arbiter loop in ExecInitPartitionInfo() is executor-internal and cannot be served from outside core, and the XXX comment confirms nothing does it today. The patch extends the existing relcache and get_partition_ancestors() path, so it adds no second mechanism. Whether other callers of get_partition_ancestors() should share the cache is a design question for another angle.
- demand: 0 objection(s). No demand objection. The patch speeds up an existing, widely used path (`insert ... on conflict` on partitioned tables) that an XXX comment already marks as slow, and Álvaro Herrera asked for the full-list design in message #4. The extra cost falls on index re-parenting during attach and detach, which hits the same partitioned-table users who gain. The material has no field reports, but this is a fix to a common path, not a new feature that needs users.

### Precedents retrieved

- P1: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P2: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant
- P3: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant; reason summarised from its thread
- P4: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant; reason summarised from its thread
- P5: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P6: [Revert "Don't lock partitions pruned by initial pruning"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1722d5eb05d8e5d2e064cd1798abcae4f296ca9d) (revert, Reverted, 2025-05-22) - judged not relevant
- P7: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P8: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P9: [refactor ExecInitPartitionInfo](https://commitfest.postgresql.org/patch/6280/) (cf, Rejected, 2026-07-06) - judged not relevant; reason summarised from its thread
- P10: [Use variadic macros for syscache lookup functions](https://commitfest.postgresql.org/patch/4261/) (cf, Withdrawn, 2023-11-14) - judged not relevant; reason summarised from its thread
- P11: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant; reason summarised from its thread
- P12: [Fix severe performance regression with gettext 0.20+ on Windows](https://commitfest.postgresql.org/patch/6491/) (cf, Returned with feedback, 2026-03-27) - judged not relevant; reason summarised from its thread
- P13: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P14: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant
- P15: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P16: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P17: [Revert incorrect memory-conservation hack in inheritance_planner().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0f36045b2e3d037bb7647d84373404fa4ba9588) (revert, Reverted, 2011-01-13) - judged not relevant
- P18: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant
- P19: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P20: [Revert "Allow ON CONFLICT .. DO NOTHING on a partitioned table."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f05230752d53c4aa74cffa9b699983bbb6bcb118) (revert, Reverted, 2017-03-31) - judged not relevant
- P21: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P22: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant
- P23: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P24: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 92 s; cost reported by the CLI: $1.31
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:57.241Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
