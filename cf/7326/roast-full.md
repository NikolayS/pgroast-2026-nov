# Design pre-review: Restructured Shared Buffer Hash Table

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7326](https://commitfest.postgresql.org/patch/7326/) (Needs review); authors: Dhruv Aron (dhruv.aron.gmail.com)
- Thread: [Restructured Shared Buffer Hash Table](https://www.postgresql.org/message-id/flat/CAAStW3JX_7LcCD1U%2B89vyMmLreDs0XsBU-JDdFf5PD_Woy1pag%40mail.gmail.com), started 2026-07-07
- Messages read: 30 of 30
- Patch prose read: lock_hold_benchmark.diff, restructured_shared_buffer_table.patch, test.diff (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The decisive objection is replication-ondisk-1. The flat table needs slot x unlinked before it is reused, message #12 rejects the spinlock-held delete that guarantees this, and no posted version carries an agreed replacement. maintenance-fit-1 adds that no check catches a later path that breaks the rule. demand-1 and prior-attempts-1 are open but do not stop the idea, since message #2 supports a separate table and the micro-benchmarks and memory saving are real.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 30 messages and no code were read, so the v6 two-step delete could not be checked against the slot-reuse race. Whether the partition lock already gives the needed exclusion on some paths could not be judged.

## The proposal, restated

**Problem.** The shared buffer mapping table uses dynahash, whose directory, segment, bucket-chain and freelist layers the author calls excessive for this one table. Each buffer hit, insert and eviction pays that cost, and the table uses more memory than needed. The target is every workload with heavy buffer lookups, and large buffer pools in particular.

**Proposed design.** buf_table.c gets its own table built on two arrays: bucket heads and entries, with entries[x] always describing buffer x. Each entry holds a BufferTag and a next index, so each bucket is a linked list over the flat arrays. Because a slot cannot hold a stale and a new entry at once, the patch moves BufTableDelete() under the buffer header spinlock in InvalidateBuffer().

**User-visible surface.**
- Shared memory allocation "Shared Buffer Lookup Table" in pg_shmem_allocations is replaced by "Shared Buffer Lookup Buckets" and "Shared Buffer Lookup Entries".
- BufTableInsert/BufTableLookup/BufTableDelete keep their role; reviewer variants add delete by buffer id.
- Benchmark test module buftable_bench with SQL functions buftable_bench_probe() and buftable_bench_collide(); x86_64 only.

**What the author claims.**
- Micro-benchmarks show lookup, insert and delete about 1.2 to 2.2 times faster than dynahash at 256MB to 16GB shared_buffers.
- Gains shrink at 16GB in the first message's test; the author links this to cache behaviour at larger sizes.
- At 128MB the table uses about half the shared memory of dynahash (Heikki's measurement).
- pgbench shows no statistically significant TPS change, with small select-only gains (author's 30-run rerun).
- A follow-up design resizes the table online with linear hashing, for the dynamic shared buffers work.

**Thread state.** Heikki, Andres, Andrey Borodin and reviewer Alexandre Felipe replied; Andres rejects holding the header spinlock across BufTableDelete() and doubts the value next to a radix-tree redesign. Alexandre posted v6 with two-step delete and dropped his lock-free variant after CFBot failed in 027_stream_regress; no fixed author version is posted.

## 2. Prior attempts

### drop/truncate table sucks for large values of shared buffers (related)

- **Link:** [https://commitfest.postgresql.org/patch/286/](https://commitfest.postgresql.org/patch/286/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)
- **Date:** decided 2015-07-02; started 2015-06-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Heikki Linnakangas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected that the approach fails badly if the caller is wrong about the relation's maximum page number (for example, kernel lseek bugs). He also noted that it does not help DROP TABLESPACE or DROP DATABASE, which also scan the whole buffer pool. Heikki Linnakangas returned it with feedback, citing the issues raised, a still-unwritten alternative (delayed cleanup during the checkpoint scan), and a preference for a structure such as a radix tree that can find all buffers for a relation or database. Tom Lane added that committing a band-aid would make more complete fixes harder to pursue.
- **Reason (quote):** "Committing a band-aid will just make it harder to pursue real fixes." ([message](https://www.postgresql.org/message-id/14561.1435845813%40sss.pgh.pa.us))
- **What is different this time:** P3 added a per-block lookup path for DROP/TRUNCATE, and this proposal replaces the table for every lookup. P3's reasons concern that path and do not transfer; the shared point is the stated preference for a radix-tree structure.


## 3. Objections a reviewer is likely to raise

### 1. Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers

fixable · level 2, rubric item 7 · design, on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-1`

InvalidateBuffer() releases buffer x, and a concurrent backend reuses slot x before the old entry is unlinked. The bucket chain breaks, so a later BufTableLookup() misses a block that is still cached. A second buffer is then loaded for the same page, one copy's changes are lost at write-out, and the loss shows up only later as heap or index corruption, on a primary or during standby replay.

Because entries[x] must describe buffer x, a stale entry and a new entry cannot exist at the same time (message #16). The v1 fix holds the buffer header spinlock across BufTableDelete(). Andres rejects that in message #12, since an error there cannot be recovered and it risks deadlock with the partition lock. The v6 two-step delete (message #18) comes with no argument for why the invariant holds, and the CFBot failures in 027_stream_regress (message #24) show that this test does catch races in this table.

**Precedents.**
- [BufferAlloc: don't take two simultaneous locks](https://commitfest.postgresql.org/patch/3483/) (Returned with feedback, 2022-10-12); thread: [link](https://www.postgresql.org/message-id/flat/1edbb61981fe1d99c3f20e3d56d6c88999f4227c.camel@postgrespro.ru)

Thread: [#12 (Andres Freund, 2026-08-11)](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#16 (Dhruv Aron, 2026-08-13)](https://www.postgresql.org/message-id/CAAStW3KnhSSMywM0kTLjOQOxhm1%2B0jFEh%2BSbZMuMytRYGzLcmg%40mail.gmail.com), [#18 (Alexandre Felipe, 2026-08-16)](https://www.postgresql.org/message-id/CAE8JnxOPr1JZ9xhhbgyAnTt3qFmtNO-OAwoxN_sq_ug1WWma9A%40mail.gmail.com), [#24 (Alexandre Felipe, 2026-09-21)](https://www.postgresql.org/message-id/CAE8JnxNGtaKUs9ZqkcPhWcEJ1RFfGZHJOAqXiqTGQzCe5gv97g%40mail.gmail.com), [#25 (Alexandre Felipe, 2026-10-11)](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com)

**Question:** Without the spinlock held across BufTableDelete(), what ordering of buffer header lock and partition lock stops slot x from being reused before its old entry is unlinked, including when an error is raised partway?

Counter-check: held. Message #16 confirms the flat table is unsafe without the spinlock-held delete, which message #12 rejects, and no posted version carries an agreed replacement. [P7] closed for lack of a rebase and must not be used as support; the CFBot failure concerned the withdrawn lock-free variant.

### 2. Slot-reuse invariant adds a silent rule for every buffer-reuse path

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a new path that evicts or reuses a buffer, for example shrinking shared buffers online, and releases the header lock before the mapping delete. A concurrent backend reuses slot x and cuts the old chain, so lookups miss cached pages and the same page can be read into a second buffer. Nothing asserts, and the damage shows up later as corruption.

Dynahash tolerates a stale entry and a new entry at once because it allocates from a freelist (message #16), but the flat table does not. v1 closes the race by holding the header spinlock across BufTableDelete(), which message #12 rejects as unsafe, since spinlocks have no error recovery and the change risks deadlock. The lock-free variant failed 027_stream_regress on CFBot (message #24) and was withdrawn (message #25). No posted version states the replacement protocol or how a violation would be detected.

Thread: [#12 (Andres Freund, 2026-08-11)](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#16 (Dhruv Aron, 2026-08-13)](https://www.postgresql.org/message-id/CAAStW3KnhSSMywM0kTLjOQOxhm1%2B0jFEh%2BSbZMuMytRYGzLcmg%40mail.gmail.com), [#24 (Alexandre Felipe, 2026-09-21)](https://www.postgresql.org/message-id/CAE8JnxNGtaKUs9ZqkcPhWcEJ1RFfGZHJOAqXiqTGQzCe5gv97g%40mail.gmail.com), [#25 (Alexandre Felipe, 2026-10-11)](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com)

**Question:** What protocol replaces the spinlock-held delete, and what check makes a future reuse path fail loudly if it breaks the entries[x] invariant?

Counter-check: held. This is distinct from replication-ondisk-1 because it concerns future buffer-reuse paths, such as the shared buffers shrink planned in message #7. The thread states no protocol, and no check that fails when a new path breaks the entries[x] invariant.

### 3. No workload-level gain shown to justify a dedicated hash table

fixable · level 1b, rubric item 4 · demand, maintenance burden ("who maintains this in 10 years"), performance on the hot path · `demand-1`

A committer pushes a custom buffer mapping table on micro-benchmark numbers. Users running pgbench-like or production workloads see no TPS change, but every later bufmgr patch has to respect the new slot-reuse invariant. The project finds out when a buffer-mapping redesign has to replace or work around it.

The only end-to-end data shows no significant TPS change: the author's 30-run rerun has confidence intervals spanning zero in most cells (message #10), and Andrey saw a possible simple-update regression (message #9). The brief targets large buffer pools, but the speedup shrinks to about 1.02x for insert and delete at 16GB (message #1, message #27). Heikki asked for real-world impact (message #2), and no field report or profile with numbers has answered it. Andres questions spending review bandwidth here instead of on the architectural problems (message #12).

**Precedents.**
- [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (Returned with feedback, 2015-07-02); thread: [link](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)

Thread: [#1 (Dhruv Aron, 2026-07-07)](https://www.postgresql.org/message-id/CAAStW3JX_7LcCD1U%2B89vyMmLreDs0XsBU-JDdFf5PD_Woy1pag%40mail.gmail.com), [#2 (Heikki Linnakangas, 2026-07-07)](https://www.postgresql.org/message-id/dbbd1998-19ff-4ac2-b4b1-a39f4ec1b0f5%40iki.fi), [#9 (Andrey Borodin, 2026-08-06)](https://www.postgresql.org/message-id/FE84530D-D06E-4317-A999-CE6F433E6CE3%40yandex-team.ru), [#10 (Dhruv Aron, 2026-08-07)](https://www.postgresql.org/message-id/CAAStW3LMwY%3DSW7GLmn81mzQGH%3DbH%2BdwZZCrsfNbM9b7eiptLjQ%40mail.gmail.com), [#12 (Andres Freund, 2026-08-11)](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#27 (Dhruv Aron, 2026-10-12)](https://www.postgresql.org/message-id/CAAStW3KitUuDbLB17N53aGj-X89unTRpV1BYYqjSc-UV%2BXz-5w%40mail.gmail.com)

**Question:** Which real workload, at which shared_buffers size, spends measurable CPU in BufTableLookup today and shows a repeatable end-to-end gain with this patch?

Counter-check: held. Heikki asked for real-world impact in message #2, and the pgbench runs in messages #9 and #10 show no significant change. The 1.02x figure at 16GB is contradicted by the later probe results (1.3 to 1.9 times), and [P3] does not bear on demand.

### 4. Tuning the hash table may delay the buffer mapping redesign

fixable · level 1b, rubric item 6 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

The flat table is committed, and the linear-hashing resize from message #7 is then built on it for dynamic shared buffers. Later, someone starts the relation-plus-block radix tree from message #12 and finds two hash-specific layers to replace. The constraint that entries[x] describes buffer x now ties bufmgr.c to the old layout.

[P3] was returned partly because a band-aid was seen to make a structure that finds all buffers of a relation harder to pursue. Andres repeats that point here in message #12: per-relation iteration, ordered lookup and 20-byte keys, plus doubt about spending effort on the current design. Message #25 answers with cost estimates for a radix tree, but the thread did not settle the point. The proposal does not say whether the flat table is a step toward the redesign or a separate path.

**Precedents.**
- [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (Returned with feedback, 2015-07-02); thread: [link](https://www.postgresql.org/message-id/flat/CAA4eK1JPLGjpMeJ5YLNE7bpNBhP2EQe_rDR+Aw3atNfj9WkAGg@mail.gmail.com)

Thread: [#7 (Dhruv Aron, 2026-07-17)](https://www.postgresql.org/message-id/CAAStW3Kq5_55X_G74P%3DaovoDBuUEeE_pionGVQ6F_dS7BThLqg%40mail.gmail.com), [#12 (Andres Freund, 2026-08-11)](https://www.postgresql.org/message-id/bk5cbayf3q4yll4fqs2wbmonzhykg43k2k5v37vokkko53ebsd%40spcp4mxlb7hb), [#25 (Alexandre Felipe, 2026-10-11)](https://www.postgresql.org/message-id/CAE8JnxOcd%2BJ0A8iM%3DBwwhovXoih5AOtvUcxPRbqgtn9CNAjPXQ%40mail.gmail.com)

**Question:** Does the flat table, with its planned resize, make the per-relation, ordered buffer mapping in message #12 any easier or harder to build later?

Counter-check: held. Andres raises this in message #12 and the author does not answer it; message #25 only gives a reviewer's cost estimates. [P3] concerned a DROP/TRUNCATE lookup path, so it must not be used as support, and an internal table can be replaced later.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers |
| WAL / replication / standby | **yes** | see objection replication-ondisk-1: Slot-reuse invariant has no agreed safe design, and a miss means duplicate buffers |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | buf_table.c gets its own hash table, and bufmgr.c gets a new ordering rule for every path that reuses a buffer. |
| performance on the hot path | **yes** | The patch rewrites BufTableLookup, BufTableInsert and BufTableDelete and changes buffer header lock hold time in InvalidateBuffer(). |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Without the spinlock held across BufTableDelete(), what ordering of buffer header lock and partition lock stops slot x from being reused before its old entry is unlinked?
2. Which patch version is under review, the v1 from message #1 or the v6 with two-step delete, and does the author agree with the v6 approach?
3. What assertion or check makes a future buffer-reuse path, such as shrinking shared buffers online, fail loudly if it breaks the entries[x] invariant?
4. Which real workload, at which shared_buffers size, shows a repeatable end-to-end gain with this patch, and is the simple-update regression from message #9 reproducible?
5. Does the flat table, with its planned linear-hashing resize, make the per-relation, ordered buffer mapping from message #12 easier or harder to build later?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas and Andres Freund replied in the thread; the owner must know bufmgr.c locking, buffer invalidation and eviction.
- **Liability:** The owner signs for the correctness of the buffer mapping under concurrency for as long as the table exists. A broken chain means two buffers for one page and lost writes, found only later as corruption.
- **Author follow-up:** The author reran pgbench with 30 runs (message #10) and offered a follow-up resize patch, but the latest versions come from the reviewer and no fixed author version is posted.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Every workload uses the lookup path, but pgbench shows no significant TPS change, so who gains measurably is not shown. The patch is small, but the review must prove a concurrency invariant in bufmgr.c, and the delete protocol is not yet agreed.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Bespoke table optimizes a design that a radix redesign would replace. Refuted (not specific): Duplicate of prior-attempts-1 for the redesign point and of demand-1 for the missing workload gain. It adds no separate failure scenario.

### Review angles

- prior-attempts: 1 objection(s). Only [P3] carries a reason that still applies. The [P7] reason is a missing rebase.
- standards-compat: 0 objection(s). The user-facing contract barely changes: no SQL syntax, GUC, catalog, dump or psql change. The only visible change, the split of the "Shared Buffer Lookup Table" row in pg_shmem_allocations into two rows, follows how shared memory allocation names already change between releases, and a committer would not block on it. The objections that matter here are about locking safety and design, which other angles cover.
- replication-ondisk: 1 objection(s). The proposal changes only shared-memory structures. It touches no on-disk format, WAL record, catalog, wire protocol or pg_upgrade path. The only exposed name change is in pg_shmem_allocations. My one objection is about an in-memory invariant whose failure leads to lost page writes.
- security: 0 objection(s). The proposal changes an internal shared memory structure and adds no privilege, role, SQL surface or new data exposure. A collision-based denial of service would need control over relfilenumbers, and dynahash has the same exposure, so I found no realistic failure scenario. The spinlock and error-recovery risks belong to the safety and design angle, and Andres raised them in message #12.
- maintenance-fit: 2 objection(s). Two objections from the maintenance angle. Hot-path cost for non-users is not a concern, since the change targets the lookup path for everyone and the measurements show it is faster.
- already-exists: 0 objection(s). No existing feature meets this need, so this angle raises nothing. The closest alternative is removing the directory indirection from dynahash for all shmem hash tables (message #2), but Heikki himself judged a separate table for buffer lookup worthwhile. A generic dynahash change also could not provide the entries[x]-is-buffer-x layout or the memory saving. No extension can replace buf_table.c, and radixtree.h belongs to the redesign Andres proposed (message #12), which is a design question for other angles.
- demand: 1 objection(s). Interest in a faster buffer lookup is real: Heikki and Andres both call dynahash a poor fit here. What is missing is evidence of a workload-level benefit.

### Precedents retrieved

- P1: [\[BUG\] Assert fails for local buffers in BufferIsExclusiveLocked](https://commitfest.postgresql.org/patch/5427/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P2: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P3: [drop/truncate table sucks for large values of shared buffers](https://commitfest.postgresql.org/patch/286/) (cf, Returned with feedback, 2015-07-02) - used in the review; reason summarised from its thread
- P4: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P5: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P6: [Auto-tune shared_buffers to use available huge pages](https://commitfest.postgresql.org/patch/6427/) (cf, Withdrawn, 2026-06-24) - judged not relevant; reason summarised from its thread
- P7: [BufferAlloc: don't take two simultaneous locks](https://commitfest.postgresql.org/patch/3483/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P8: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P9: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P10: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant; reason summarised from its thread
- P11: [revert to showing buffer counts in explain (buffers)](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=27cb626f7a28abd18c0e8a34fc8d6674b0f6cdf3) (revert, Reverted, 2010-02-16) - judged not relevant
- P12: [Allow shared buffer cache contents to affect index cost computations](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P13: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant; reason summarised from its thread
- P14: [Subtrans performance smoothing](https://commitfest.postgresql.org/patch/3792/) (cf, Withdrawn, 2022-09-06) - judged not relevant
- P15: [KnownAssignedXidsGetAndSetXmin performance](https://commitfest.postgresql.org/patch/3271/) (cf, Returned with feedback, 2022-11-15) - judged not relevant
- P16: [Precompute string lengths in PerformRadiusTransaction](https://commitfest.postgresql.org/patch/6377/) (cf, Withdrawn, 2026-03-25) - judged not relevant
- P17: [Revert "Fix link error for test_radixtree module on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6d9751fa8fd40c988541c9c72ac7a2095ba73c19) (revert, Reverted, 2024-03-08) - judged not relevant
- P18: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant
- P19: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant
- P20: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant
- P21: [Replace spinlock with lwlock in pg_stat_statements](https://commitfest.postgresql.org/patch/2634/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P22: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P23: [Improve documentation for spinlocks file](https://commitfest.postgresql.org/patch/208/) (cf, Rejected, 2015-07-02) - judged not relevant
- P24: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 146 s; cost reported by the CLI: $4.12
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:41:12.658Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
