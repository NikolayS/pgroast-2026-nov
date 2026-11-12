# Design pre-review: Reducing relcache memory usage 2: shrink sizeof(RelationData)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7483](https://commitfest.postgresql.org/patch/7483/); [thread](https://www.postgresql.org/message-id/flat/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), 18 of 18 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and the reviewers in the thread treat it as worth doing. The surviving objections are all fixable design points. maintenance-fit-1 is the main one: nothing fails loudly when code reads the inactive union member. prior-attempts-1, maintenance-fit-2 and maintenance-fit-3 ask for v6 measurements of memory and of the relkind check cost.

Not checked: Without the code, the audit of union member reads and the correctness of on-demand lock identifiers in every caller could not be checked. Only 3 of 18 messages were read in full, so later agreements may not be reflected.

## Prior attempts

- [Change the bool member of the Query structure to bits](https://www.postgresql.org/message-id/flat/d05a1c87-82e9-49a3-a4e2-e0d2b38b2e8b@yeah.net) (similar idea; Withdrawn, 2024-02-20). "The amount of space saved is quite negligible. If queries had many Query structs then it could matter, but they don't." ([message](https://www.postgresql.org/message-id/184953.1708443951%40sss.pgh.pa.us)) Different now: P11 shrank Query, which exists in small numbers and still rounded up to the same chunk size. This proposal targets RelationData, which can exist hundreds of thousands of times per backend, and uses a slab context so every saved byte counts.
- [Separate memory contexts for relcache and catcache](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com) (related; Returned with feedback, 2024-12-11). "Andres's concern, IIUC, is that the additional memory contexts will cause additional fragmentation." ([message](https://www.postgresql.org/message-id/4edadffe14ead8860f4d02141aaf154871fef0cb.camel%40j-davis.com)) Different now: P1 split CacheMemoryContext into per-cache child contexts for accounting. This proposal adds one slab child context for RelationData to save memory, so the fragmentation concern applies only in part.

## Objections a reviewer is likely to raise

### 1. Union reads of the wrong member fail silently in future patches

A later core patch or an extension reads rel->rd_rules, rd_indexprs or rd_tableam without first checking the relation kind. It compiles and passes tests on plain tables, then crashes or returns a wrong-typed pointer when an index, or a table with rules, reaches that path in production.

Reviewers found three such crashes after the patch passed check-world: predicate locking (message #6), the NIL contract of RelationGetIndexExpressions() (message #11), and relation_is_updatable() (message #17). The only audit method stated is grep plus tests (message #3). The discriminant field that could back assertion-checked accessors exists only as a draft top commit (message #15). Today nothing fails loudly when a future reader of RelationData forgets the kind check.

**Question:** How will a cassert build catch any access to the inactive union member, in core and in extensions, without relying on a reviewer to spot it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/d655cd02-ce3e-4e04-8035-c70d1464deb3%40gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29GSHtWSoWpf71Mrv768tZN8d-5S5jsS5-r_Wn68dV51ww%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CABXr29GTZkymnuJ6aLVeW36Ysma7j9%2BUuz05Eb3uTsV8G5MaHg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/d88fa093-a42b-444b-b640-6adb8c4508d2%40gmail.com), [#17](https://www.postgresql.org/message-id/632af30c.4d4c.1a1581940a9.Coremail.jiezhilove%40126.com); `maintenance-fit-1`</sub>

### 2. Slab context added without v6 measurements of net memory savings

A site with many backends and small schemas upgrades. Each backend now holds a slab context for RelationData, and maybe a second one for RelationPartitionInfo, with partly filled blocks after relcache invalidations. Per-backend memory may not drop, or may rise, and nobody sees it until after release.

In [P1], extra cache contexts were held back over the concern that "the additional memory contexts will cause additional fragmentation." v6-0002 adds such a context, and message #14 suggests another for RelationPartitionInfo. The only memory numbers are from v1 under aset (message #1). Message #6 measured a slab setup only for the dropped 0007/0008. Matthias proposed the slab context in message #2, which lowers the concern but does not measure it.

Precedent: [Separate memory contexts for relcache and catcache](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com) (Returned with feedback, 2024-12-11)

**Question:** What does CacheMemoryContext plus the new slab context use under v6, for both a small schema and the 187k-relation schema, including after invalidation churn?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), [#2](https://www.postgresql.org/message-id/CAEze2Wj3rq6V%2BkAoXS80oTPi4s1GpsH4Gx7h-jqBBLzA64CyPQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29GSHtWSoWpf71Mrv768tZN8d-5S5jsS5-r_Wn68dV51ww%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com); `prior-attempts-1`</sub>

### 3. Loading partition info on demand may cost partitioned schemas more memory

A user with tens of thousands of partitions loads them into the relcache. Each partition that caches its partition qual now allocates a separate RelationPartitionInfo, rounded up to a 128-byte aset chunk. That is more than the bytes the move saves in RelationData, so memory per backend can go up.

Matthias van de Meent notes in message #14 that RelationPartitionInfo is just over the 64-byte aset bucket and asks for a slab context or a smaller struct. The reply in message #16 offers options but no decision. Large partitioned schemas are a main target of this work. The only memory numbers come from v1, before this split, on a schema of plain tables and indexes (message #1).

**Question:** What is the CacheMemoryContext size with v6 on a schema with many partitions, compared with master?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), [#14](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/3c465219-a4fe-4ed4-b921-86b0cec2782c%40gmail.com); `maintenance-fit-2`</sub>

### 4. Extra indirection on common relkind checks is not measured where it lands

Every caller that used to test rd_index now follows rd_rel to read relkind, including PredicateLockTID() for non-index relations. A SERIALIZABLE or index-heavy workload pays an extra dependent load on each call, and nobody sees it until a later benchmark.

Matthias van de Meent disputes in message #14 that the cost is minimal, and points to the number of rd_index checks that became RelationIsIndex() as a sign of a common pattern. The only benchmark is a single-client pgbench -S run on partitioned tables (message #4), which does not use SERIALIZABLE predicate locking. The draft variant field in message #15 could remove the indirection, but it is not yet part of the series.

**Question:** Can you show measurements of the paths that switched to RelationIsIndex(), such as SERIALIZABLE workloads, with and without an in-struct variant field?

<sub>fixable, level 2, performance on the hot path; messages [#4](https://www.postgresql.org/message-id/e8c52d3a-b35b-4351-9896-12310323e465%40googlemail.com), [#7](https://www.postgresql.org/message-id/6d939520-6877-4aab-88b2-2d0b37c66419%40gmail.com), [#14](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/d88fa093-a42b-444b-b640-6adb8c4508d2%40gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. How will a cassert build catch a read of the inactive union member, in core and in extensions, without relying on a reviewer to spot it?
2. What do CacheMemoryContext and the new slab context use under v6, for a small schema and for the 187,000-relation schema, including after invalidation churn?
3. What is the relcache memory use with v6 compared with master on a schema with many partitions that cache a RelationPartitionInfo?
4. What do SERIALIZABLE and index-heavy workloads show for the paths that switched to RelationIsIndex(), with and without the in-struct variant field?
5. How do extension authors learn that fields such as rd_index and rd_rules now share storage and need a relation kind check first?

## Touch points

- **on-disk format / pg_upgrade:** The patch bumps RELCACHE_INIT_FILEMAGIC, so old pg_internal.init files are rebuilt.
- **WAL / replication / standby:** The lock identifier change touches xlogutils.c, which builds fake relcache entries during WAL replay.
- **ABI / extension API:** RelationData layout changes, moves to rel_internal.h, loses rd_lockInfo, and overlays table-only and index-only fields in a union.
- **security / privileges:** In v5 any user could crash a backend with pg_relation_is_updatable() on an index (message #17); v6 fixes it.
- **backpatchability:** Renamed, moved and overlaid RelationData fields make later fixes in relcache callers harder to back-patch.
- **maintenance burden ("who maintains this in 10 years"):** Every future reader of a union member must check the relation kind first, and nothing enforces it.
- **performance on the hot path:** RelationIsIndex() reads rd_rel->relkind where callers used to test rd_index, and lock identifiers are computed on each use.

## Who would commit it, and when

- Owner: silent. The CommitFest entry lists a reviewer but no committer. The owner needs to know relcache.c, its invalidation and init-file paths, and the lock manager. Álvaro Herrera commented on the header move.
- Priority: proceed; audience a sizeable minority; review cost high. Installations with very large schemas gain about 10% of CacheMemoryContext per backend by the v1 numbers. The series touches relcache, lmgr, access methods and a widely included header, and the union needs a careful audit. Review is active and the open points are fixable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 7.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
