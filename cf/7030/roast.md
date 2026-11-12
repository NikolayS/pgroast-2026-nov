# Design pre-review: Caching a partition index's parent OID in the relcache?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7030](https://commitfest.postgresql.org/patch/7030/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWUV3Tz5A161y2EcCMqnmCYqKNm6Jt%3DrXCx9bEOJ-hqL%3DA%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, measured, and already flagged by an XXX comment in the code. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about where the subtree invalidation rule lives and what enforces it. A committer who knows relcache invalidation has to judge that rule, together with the unanswered locking question.

Not checked: Without the code, this review could not check whether the cache is reset on every relcache path, whether the NoLock subtree scan is safe, or whether index_concurrently_swap() invalidates descendants. Only 3 of 7 thread messages were shown.

## Prior attempts

None found before 2026-07-17.

## Objections a reviewer is likely to raise

### 1. Cache correctness depends on one function remembering to invalidate descendants

A later patch adds a new path that changes an intermediate partitioned index's pg_inherits link without going through IndexSetParentIndex(). Leaf indexes keep stale ancestor lists. Users find out when upserts fail with "invalid arbiter index list" or resolve the wrong arbiter, with no assert or test failure to catch it earlier.

Message #3 shows that pg_inherits changes send no relcache invalidation. So v2's correctness rests on a new rule: whoever re-parents an index must invalidate the whole descendant subtree, and that rule lives only inside IndexSetParentIndex(). The proposal does not say how a future writer of pg_inherits for indexes would learn this rule, or what would fail if they missed it. The single regression test covers only the ATTACH/DETACH path, and the question in message #5 about other paths has no answer.

**Question:** Which code paths can change a partitioned index's pg_inherits link today, and what would make a future path that skips the subtree invalidation fail loudly?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/CAJTYsWXquWQR2cY9jjfSjG7nFHhTD_GW1hFO6xnQSXBtfNk3tA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAJTYsWWEDMpFETWC1AE%3Dr%2Bos__R9FbK5ELFJBKEKhT%2BuuWtjEw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJTYsWX-9HMD3T43txYdCubXk0FaBDMVygU3c6C3zDMqv0xDOw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which code paths can change a partitioned index's pg_inherits link today, and what would make a future path that skips the subtree invalidation fail loudly?
2. Is NoLock in find_all_inheritors() safe on every path that calls IndexSetParentIndex(), including paths other than ATTACH and DETACH?
3. Does REINDEX CONCURRENTLY, which moves the pg_inherits link in index_concurrently_swap(), still invalidate the cached ancestor lists of descendant indexes in v2?
4. How much memory do the cached ancestor arrays use in a backend that opens many partition indexes in a hierarchy of three or more levels?
5. Can other callers of get_partition_ancestors() use the new cache, or does get_partition_index_ancestors() serve only the arbiter loop?

## Touch points

- **catalog changes:** No catalog schema change, but the cache depends on pg_inherits rows, whose changes send no relcache invalidation.
- **ABI / extension API:** Adds rd_partancestorcount and rd_partancestors to RelationData and get_partition_index_ancestors() to catalog/partition.h.
- **maintenance burden ("who maintains this in 10 years"):** Every future path that re-parents a partitioned index must invalidate the descendant-index subtree, and only IndexSetParentIndex() does so.
- **performance on the hot path:** Changes the arbiter-mapping loop in ExecInitPartitionInfo(); ATTACH plus DETACH with 256 leaf subpartitions slows from about 10 ms to 31 ms.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera proposed the full-list design in message #2 and, per the brief, supported the subtree invalidation. The owner must know relcache invalidation and partitioned-index DDL.
- Priority: proceed; audience a sizeable minority; review cost medium. The patch gives 16% to 31% more TPS for repeated upserts on partitioned tables with several indexes per leaf, and a committer shaped the design. The patch is small (about 17 KB) but changes relcache invalidation, and the code was not read here.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
