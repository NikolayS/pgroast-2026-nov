# Design pre-review: Reducing relcache memory usage 2: shrink sizeof(RelationData)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7483](https://commitfest.postgresql.org/patch/7483/) (Needs review); authors: David Geier (geidav)
- Thread: [Reducing relcache memory usage 2: shrink sizeof(RelationData)](https://www.postgresql.org/message-id/flat/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), started 2026-09-10
- Messages read: 18 of 18
- Patch prose read: v6-0001-Assert-CacheMemoryContext-exists-in-relcache-call.patch, v6-0002-Allocate-RelationData-structs-in-a-dedicated-slab.patch, v6-0003-Move-RelationData-to-an-internal-header.patch, v6-0004-Overlay-relation-kind-specific-relcache-fields.patch, v6-0005-Allocate-partition-cache-data-lazily.patch, v6-0006-Pack-RelationData-to-eliminate-padding.patch, v6-0007-Compute-relation-lock-identifiers-on-demand.patch, v1-0001-Move-RelationData-to-new-include.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and the reviewers in the thread treat it as worth doing. The surviving objections are all fixable design points. maintenance-fit-1 is the main one: nothing fails loudly when code reads the inactive union member. prior-attempts-1, maintenance-fit-2 and maintenance-fit-3 ask for v6 measurements of memory and of the relkind check cost.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the audit of union member reads and the correctness of on-demand lock identifiers in every caller could not be checked. Only 3 of 18 messages were read in full, so later agreements may not be reflected.

## The proposal, restated

**Problem.** Each relcache entry costs 520 bytes because RelationData is 488 bytes and aset rounds it up to a 512-byte chunk. With many tables and indexes, CacheMemoryContext grows large in every backend. The patch targets installations with large schemas.

**Proposed design.** RelationData moves to a dedicated slab context, so every byte saved counts. The patch then overlays table-only and index-only fields in an anonymous union, moves partition state into a lazily allocated RelationPartitionInfo, reorders members to remove padding, and computes the lock identifier on demand instead of caching rd_lockInfo. sizeof(RelationData) goes from 496 to 248 bytes on Linux x86-64.

**User-visible surface.**
- C API: RelationData layout changes; table-only and index-only fields share storage in an anonymous union.
- C API: RelationData definition moves from utils/rel.h to new utils/rel_internal.h (under discussion).
- C API: partition fields such as rd_partdesc move into a separate RelationPartitionInfo struct reached by a pointer.
- C API: rd_lockInfo is removed; RelationGetLockRelId() computes it; new RelationIsIndex() macro and AllocRelationData() allocator.
- File format: RELCACHE_INIT_FILEMAGIC is bumped, so old pg_internal.init files are rebuilt.

**What the author claims.**
- CacheMemoryContext drops from about 589 MiB to 536 MiB, about 10%, with 38,966 tables and 148,468 indexes in one backend.
- A pgbench run with 1000 partitions showed no measurable TPS regression from the partition info change.
- The extra indirection through rd_rel->relkind costs nothing relative to other work.
- Computing the lock identifier on demand is more correct after loading a pg_internal.init copied by CREATE DATABASE.
- The union, packing and header changes are mechanical.

**Thread state.** Matthias van de Meent reviews in depth; he objects to extra indirection in RelationIsIndex() and wants a variant field in the struct, now a draft top commit. Haibo Yan and Yilin Zhang found union-related crashes, all fixed in v6; Álvaro Herrera's and the gen_node_support.pl approach to the header move is still open.

## 2. Prior attempts

### Change the bool member of the Query structure to bits (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4844/](https://commitfest.postgresql.org/patch/4844/)
- **Thread:** [https://www.postgresql.org/message-id/flat/d05a1c87-82e9-49a3-a4e2-e0d2b38b2e8b@yeah.net](https://www.postgresql.org/message-id/flat/d05a1c87-82e9-49a3-a4e2-e0d2b38b2e8b@yeah.net)
- **Date:** decided 2024-02-20; started 2024-02-20
- **Outcome:** Withdrawn (cf)
- **Who objected:** Tomas Vondra, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tomas Vondra and Tom Lane found the memory saving negligible: Query shrinks from about 256B to about 232B, but allocation rounds up to a 256B chunk anyway, and a query has few Query structs. Tom Lane also objected that the code churn would make back-patching harder and that bit flags would make pprint output unreadable. The author withdrew the patch.
- **Reason (quote):** "The amount of space saved is quite negligible. If queries had many Query structs then it could matter, but they don't." ([message](https://www.postgresql.org/message-id/184953.1708443951%40sss.pgh.pa.us))
- **What is different this time:** P11 shrank Query, which exists in small numbers and still rounded up to the same chunk size. This proposal targets RelationData, which can exist hundreds of thousands of times per backend, and uses a slab context so every saved byte counts.

### Separate memory contexts for relcache and catcache (related)

- **Link:** [https://commitfest.postgresql.org/patch/4554/](https://commitfest.postgresql.org/patch/4554/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com)
- **Date:** decided 2024-12-11; started 2023-08-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Jeff Davis, Rahila Syed (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund's messages are not shown, but Jeff Davis reports Andres's concern that the extra contexts add memory fragmentation in each backend. Jeff saw rough consensus only for CatCacheContext and RelCacheContext, and the thread was still looking for ways to reduce the fragmentation. Rahila Syed and Ashutosh Bapat found little value in PlanCacheContext and SPICacheContext, which hold no allocations of their own. Michael Paquier marked the entry returned with feedback, citing the open feedback and concerns.
- **Reason (quote):** "Andres's concern, IIUC, is that the additional memory contexts will cause additional fragmentation." ([message](https://www.postgresql.org/message-id/4edadffe14ead8860f4d02141aaf154871fef0cb.camel%40j-davis.com))
- **What is different this time:** P1 split CacheMemoryContext into per-cache child contexts for accounting. This proposal adds one slab child context for RelationData to save memory, so the fragmentation concern applies only in part.


## 3. Objections a reviewer is likely to raise

### 1. Union reads of the wrong member fail silently in future patches

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later core patch or an extension reads rel->rd_rules, rd_indexprs or rd_tableam without first checking the relation kind. It compiles and passes tests on plain tables, then crashes or returns a wrong-typed pointer when an index, or a table with rules, reaches that path in production.

Reviewers found three such crashes after the patch passed check-world: predicate locking (message #6), the NIL contract of RelationGetIndexExpressions() (message #11), and relation_is_updatable() (message #17). The only audit method stated is grep plus tests (message #3). The discriminant field that could back assertion-checked accessors exists only as a draft top commit (message #15). Today nothing fails loudly when a future reader of RelationData forgets the kind check.

Thread: [#3 (David Geier, 2026-09-18)](https://www.postgresql.org/message-id/d655cd02-ce3e-4e04-8035-c70d1464deb3%40gmail.com), [#6 (Haibo Yan, 2026-10-14)](https://www.postgresql.org/message-id/CABXr29GSHtWSoWpf71Mrv768tZN8d-5S5jsS5-r_Wn68dV51ww%40mail.gmail.com), [#11 (Haibo Yan, 2026-10-17)](https://www.postgresql.org/message-id/CABXr29GTZkymnuJ6aLVeW36Ysma7j9%2BUuz05Eb3uTsV8G5MaHg%40mail.gmail.com), [#15 (David Geier, 2026-10-19)](https://www.postgresql.org/message-id/d88fa093-a42b-444b-b640-6adb8c4508d2%40gmail.com), [#17 ("Yilin Zhang", 2026-10-20)](https://www.postgresql.org/message-id/632af30c.4d4c.1a1581940a9.Coremail.jiezhilove%40126.com)

**Question:** How will a cassert build catch any access to the inactive union member, in core and in extensions, without relying on a reviewer to spot it?

Counter-check: held. Reviewers found three wrong-member reads after check-world passed, and the old idiom `rd_index != NULL` still compiles. The variant field that could back assert-checked accessors is only a draft top commit in message #15.

### 2. Slab context added without v6 measurements of net memory savings

fixable · level 2, rubric item 8 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `prior-attempts-1`

A site with many backends and small schemas upgrades. Each backend now holds a slab context for RelationData, and maybe a second one for RelationPartitionInfo, with partly filled blocks after relcache invalidations. Per-backend memory may not drop, or may rise, and nobody sees it until after release.

In [P1], extra cache contexts were held back over the concern that "the additional memory contexts will cause additional fragmentation." v6-0002 adds such a context, and message #14 suggests another for RelationPartitionInfo. The only memory numbers are from v1 under aset (message #1). Message #6 measured a slab setup only for the dropped 0007/0008. Matthias proposed the slab context in message #2, which lowers the concern but does not measure it.

**Precedents.**
- [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (Returned with feedback, 2024-12-11); thread: [link](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com)

Thread: [#1 (David Geier, 2026-09-10)](https://www.postgresql.org/message-id/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), [#2 (Matthias van de Meent, 2026-09-14)](https://www.postgresql.org/message-id/CAEze2Wj3rq6V%2BkAoXS80oTPi4s1GpsH4Gx7h-jqBBLzA64CyPQ%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-14)](https://www.postgresql.org/message-id/CABXr29GSHtWSoWpf71Mrv768tZN8d-5S5jsS5-r_Wn68dV51ww%40mail.gmail.com), [#14 (Matthias van de Meent, 2026-10-19)](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com)

**Question:** What does CacheMemoryContext plus the new slab context use under v6, for both a small schema and the 187k-relation schema, including after invalidation churn?

Counter-check: held. No memory numbers exist for v6 with the slab context, and message #6 measured slab only on v3 with 8,731 relations. P1 is weak support, and the small-schema rise is bounded by about one partly filled slab block per backend.

### 3. Loading partition info on demand may cost partitioned schemas more memory

fixable · level 2, rubric item 4 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A user with tens of thousands of partitions loads them into the relcache. Each partition that caches its partition qual now allocates a separate RelationPartitionInfo, rounded up to a 128-byte aset chunk. That is more than the bytes the move saves in RelationData, so memory per backend can go up.

Matthias van de Meent notes in message #14 that RelationPartitionInfo is just over the 64-byte aset bucket and asks for a slab context or a smaller struct. The reply in message #16 offers options but no decision. Large partitioned schemas are a main target of this work. The only memory numbers come from v1, before this split, on a schema of plain tables and indexes (message #1).

Thread: [#1 (David Geier, 2026-09-10)](https://www.postgresql.org/message-id/f97de758-b816-47db-96e0-0e6e2c2d3eef%40gmail.com), [#14 (Matthias van de Meent, 2026-10-19)](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com), [#16 (David Geier, 2026-10-19)](https://www.postgresql.org/message-id/3c465219-a4fe-4ed4-b921-86b0cec2782c%40gmail.com)

**Question:** What is the CacheMemoryContext size with v6 on a schema with many partitions, compared with master?

Counter-check: held. Message #14 raises that RelationPartitionInfo falls just past the 64-byte aset bucket, and message #16 lists options without choosing one. The thread has no memory numbers for a schema with many partitions.

### 4. Extra indirection on common relkind checks is not measured where it lands

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-3`

Every caller that used to test rd_index now follows rd_rel to read relkind, including PredicateLockTID() for non-index relations. A SERIALIZABLE or index-heavy workload pays an extra dependent load on each call, and nobody sees it until a later benchmark.

Matthias van de Meent disputes in message #14 that the cost is minimal, and points to the number of rd_index checks that became RelationIsIndex() as a sign of a common pattern. The only benchmark is a single-client pgbench -S run on partitioned tables (message #4), which does not use SERIALIZABLE predicate locking. The draft variant field in message #15 could remove the indirection, but it is not yet part of the series.

Thread: [#4 (David Geier, 2026-10-05)](https://www.postgresql.org/message-id/e8c52d3a-b35b-4351-9896-12310323e465%40googlemail.com), [#7 (David Geier, 2026-10-16)](https://www.postgresql.org/message-id/6d939520-6877-4aab-88b2-2d0b37c66419%40gmail.com), [#14 (Matthias van de Meent, 2026-10-19)](https://www.postgresql.org/message-id/CAEze2WgkvPVqLd8ZPEY_THfsqZ7XaSTBvrstEKyRXdMOzrMuKg%40mail.gmail.com), [#15 (David Geier, 2026-10-19)](https://www.postgresql.org/message-id/d88fa093-a42b-444b-b640-6adb8c4508d2%40gmail.com)

**Question:** Can you show measurements of the paths that switched to RelationIsIndex(), such as SERIALIZABLE workloads, with and without an in-struct variant field?

Counter-check: held. The engaged reviewer disputes the cost claim in message #14, and the pgbench -S run in message #4 does not use predicate locking. The variant field that would remove the indirection is not yet part of the series.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch bumps RELCACHE_INIT_FILEMAGIC, so old pg_internal.init files are rebuilt. |
| WAL / replication / standby | **yes** | The lock identifier change touches xlogutils.c, which builds fake relcache entries during WAL replay. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | RelationData layout changes, moves to rel_internal.h, loses rd_lockInfo, and overlays table-only and index-only fields in a union. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | In v5 any user could crash a backend with pg_relation_is_updatable() on an index (message #17); v6 fixes it. |
| backpatchability | **yes** | Renamed, moved and overlaid RelationData fields make later fixes in relcache callers harder to back-patch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future reader of a union member must check the relation kind first, and nothing enforces it. |
| performance on the hot path | **yes** | RelationIsIndex() reads rd_rel->relkind where callers used to test rd_index, and lock identifiers are computed on each use. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How will a cassert build catch a read of the inactive union member, in core and in extensions, without relying on a reviewer to spot it?
2. What do CacheMemoryContext and the new slab context use under v6, for a small schema and for the 187,000-relation schema, including after invalidation churn?
3. What is the relcache memory use with v6 compared with master on a schema with many partitions that cache a RelationPartitionInfo?
4. What do SERIALIZABLE and index-heavy workloads show for the paths that switched to RelationIsIndex(), with and without the in-struct variant field?
5. How do extension authors learn that fields such as rd_index and rd_rules now share storage and need a relation kind check first?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The CommitFest entry lists a reviewer but no committer. The owner needs to know relcache.c, its invalidation and init-file paths, and the lock manager. Álvaro Herrera commented on the header move.
- **Liability:** The committer signs for every wrong-member read of the union in core and in extensions, with no time limit. The committer also owns the lock identifier change in WAL replay and any memory regression from the slab context.
- **Author follow-up:** Good sign: the author posted six versions in about two months and fixed every crash that reviewers reported by v6.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Installations with very large schemas gain about 10% of CacheMemoryContext per backend by the v1 numbers. The series touches relcache, lmgr, access methods and a widely included header, and the union needs a careful audit. Review is active and the open points are fixable.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Union keeps old field names, so extension misuse compiles silently. Refuted (not specific): Duplicate of maintenance-fit-1, which covers both core and out-of-core readers of the inactive union member and asks for the same loud failure.
- `replication-ondisk-1` Union fields keep old names, so stale extension code compiles and misreads. Refuted (not specific): Duplicate of maintenance-fit-1.
- `security-1` Untagged union lets SQL-reachable code read pointers of the wrong type. Refuted (not specific): Duplicate of maintenance-fit-1. The crashes it cites from messages #6, #11 and #17 are fixed in v6, and what is left is the same missing check on reads of the inactive union member.
- `demand-1` A 10% gain on one schema may not justify an API cost for everyone. Refuted (not specific): Duplicate of maintenance-fit-1 for the misread risk and of prior-attempts-1 for the missing v6 numbers. On demand, message #1 cites a second relcache memory thread, and no reviewer questions the goal.

### Review angles

- prior-attempts: 1 objection(s). The closest precedent, P11, died because the savings were negligible. That reason does not apply here: the author reports about 10% of CacheMemoryContext saved on a large schema, and the slab context removes the rounding problem. Message #1 points to a related relcache deduplication thread, but the thread does not report a rejection there.
- standards-compat: 1 objection(s). No SQL, GUC, pg_dump or psql surface changes. The pg_internal.init magic bump rebuilds old files, and removed fields such as rd_lockInfo fail loudly at compile time.
- replication-ondisk: 1 objection(s). The pg_internal.init change is covered by the RELCACHE_INIT_FILEMAGIC bump. The fake-relcache dbOid case in WAL replay was settled in messages #5 and #7. The nbtree VACUUM invariant does not apply, because nbtutils.c changes only for lock identifiers. There is no WAL, catalog or wire protocol change.
- security: 1 objection(s). The patch adds no privilege, role or new exposure. The only security concern is the risk that an unprivileged user can crash the server through wrong-member reads of the union.
- maintenance-fit: 3 objection(s). The nbtree VACUUM bookkeeping invariant does not apply: nbtutils.c changes only to compute lock identifiers. Demand is plausible for large schemas, so no fit objection is raised.
- already-exists: 0 objection(s). Postgres has no feature, setting or extension that shrinks relcache entries; only core can change the layout of RelationData. Pooling cuts the number of backends, not the cost per entry, and the slab context reuses an existing allocator, so this angle finds nothing to raise.
- demand: 1 objection(s). Relcache growth with large schemas is a real, recurring problem, and several reviewers engaged in depth. The objection above asks for current numbers and broader evidence, not for rejection.

### Precedents retrieved

- P1: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - used in the review; reason summarised from its thread
- P2: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant
- P3: [Reorganize CacheMemoryContext](https://commitfest.postgresql.org/patch/5337/) (cf, Withdrawn, 2024-10-29) - used in the review; reason summarised from its thread
- P4: [Revert: Allow table AM to store complex data structures in rd_amcache](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=922c4c461d213a422ee7eb6c38e399607539210a) (revert, Reverted, 2024-04-11) - judged not relevant
- P5: [Revert patch that removed BUFSIZ usage. The memory has to hold the structures plus pointers used by the structure.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1ad687827607ee02a51182ea8a255c230e61c740) (revert, Reverted, 2004-09-28) - judged not relevant
- P6: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P7: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P8: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P9: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P11: [Change the bool member of the Query structure to bits](https://commitfest.postgresql.org/patch/4844/) (cf, Withdrawn, 2024-02-20) - used in the review; reason summarised from its thread
- P12: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - used in the review; reason summarised from its thread
- P13: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P14: [replace magic num in struct cachedesc with CATCACHE_MAXKEYS](https://commitfest.postgresql.org/patch/5186/) (cf, Withdrawn, 2024-08-24) - judged not relevant
- P15: [Temporary tables that do not bloat pg_catalog (a.k.a fast temp tables)](https://commitfest.postgresql.org/patch/688/) (cf, Rejected, 2016-09-20) - judged not relevant
- P16: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - judged not relevant
- P17: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant
- P18: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P19: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Union Replacement of OR logic](https://commitfest.postgresql.org/patch/4715/) (cf, Rejected, 2024-07-24) - judged not relevant
- P21: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P22: [Shared system catalog cache](https://commitfest.postgresql.org/patch/2326/) (cf, Returned with feedback, 2020-02-01) - used in the review
- P23: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P24: [Remove unused #include's](https://commitfest.postgresql.org/patch/6112/) (cf, Withdrawn, 2025-10-15) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0002-Use-union.patch not read
- attachment limit (8) reached; v1-0003-Packing-RelationData.patch not read
- attachment limit (8) reached; v1-0004-Remove-rd_lockinfo.patch not read
- attachment limit (8) reached; v1-0005-Move-out-partition-members.patch not read
- attachment limit (8) reached; v1-0006-Remove-rd_fkeyvalid.patch not read
- attachment limit (8) reached; v1-0007-Remove-rd_index.patch not read
- attachment limit (8) reached; v1-0008-Add-static-assert-for-size.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 177 s; cost reported by the CLI: $3.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 92 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:00:15.246Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
