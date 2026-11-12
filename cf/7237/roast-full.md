# Design pre-review: Reducing relcache memory usage: deduping index shapes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7237](https://commitfest.postgresql.org/patch/7237/) (Needs review); authors: Matthias van de Meent (mmeent)
- Thread: [Reducing relcache memory usage: deduping index shapes](https://www.postgresql.org/message-id/flat/CAEze2WiPyruOtUOSyRUV8mQssjmYwno0M6hkxC_iUpH-%3DW8WcA%40mail.gmail.com), started 2026-08-31
- Messages read: 31 of 31
- Patch prose read: v5-0001-rel.h-const-ify-opclass-related-fields-of-Relatio.patch, v5-0003-Deduplicate-some-index-attributes-in-the-relcache.patch, v5-0005-Relcache-Use-System-contexts-for-index-info-and-i.patch, v5-0006-Relcache-Reduce-number-of-allocations-into-shape-.patch, v5-0004-MemCTX-Add-minimal-system-context-type.patch, v5-0002-relcache-refactor-RelationInitIndexAccessInfo.patch, v1-0003-Deduplicate-some-index-attributes-in-the-relcache.patch, v1-0004-MemCTX-Add-minimal-proxy-context-type-that-just-d.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal has shown demand and no objection survived at levels 1a or 1b. Two fixable level 2 points remain. replication-ondisk-1: message #9 accepts the dangling fn_mcxt problem in shared rd_supportinfo, and no changelist through v5 says it is fixed. maintenance-fit-2: the 0003 commit message admits extra load work, and the thread has only memory numbers. Both need an answer from the author and a committer who knows relcache and fmgr.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it is not possible to check whether v5 already fixes the fn_mcxt lifetime, how the shape cache frees entries, or what the fmgr.h and c.h changes do.

## The proposal, restated

**Problem.** Each index relcache entry allocates its own opclass-derived arrays, including nkeyatts * amsupport FmgrInfos, even when many indexes share the same key definition. Each entry also gets its own aset context with large fixed overhead. This costs a lot of backend memory with many indexes, for example partitioned tables or schemas with hundreds of thousands of indexes.

**Proposed design.** A per-backend "Relation shape cache" shares one set of rd_opfamily, rd_opcintype, rd_support and rd_supportinfo among indexes with the same AM, number of key attributes, opclasses and opclass options. A new SystemContext type, by Andres Freund and first named Proxy, sends every palloc to malloc and has a small header. The relcache uses it for "index info" and "index shape" contexts.

**User-visible surface.**
- New memory context type SystemContext (systemctx.c), shown as a new type in pg_backend_memory_contexts.
- New C API for creating System contexts, available to extensions.
- New "Relation shape cache" context and per-shape "index shape" contexts under CacheMemoryContext.
- rd_opfamily, rd_opcintype and rd_support in RelationData become const, and their memory is shared across indexes; extensions that write to them break.
- fmgr.h and c.h change; the material does not say how.

**What the author claims.**
- On a freshly initdb'ed catalog, "index info" memory drops from about 288 kB to about 45 kB with the full patchset.
- An average "index info" context uses about 329 bytes, against 2086 bytes on master.
- A reviewer measured 776 MiB to 633 MiB of relcache memory on a schema with about 39,000 tables and 148,000 indexes.
- Loading index relcache entries costs slightly more work, which the author calls a worthy trade-off.
- The relcache code changes rarely, so the patch could be applied to older versions for testing.

**Thread state.** David Rowley reviewed the System context several times and sent a fixup on v4; his choice between two fixes for the realloc failure path is open. Chao Li's report of shared rd_supportinfo keeping a dangling fn_mcxt was acknowledged, and no later changelist says it was fixed.

## 2. Prior attempts

### Separate memory contexts for relcache and catcache (related)

- **Link:** [https://commitfest.postgresql.org/patch/4554/](https://commitfest.postgresql.org/patch/4554/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com)
- **Date:** decided 2024-12-11; started 2023-08-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Jeff Davis, Rahila Syed (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andres Freund's messages are not shown, but Jeff Davis reports Andres's concern that the extra contexts add memory fragmentation in each backend. Jeff saw rough consensus only for CatCacheContext and RelCacheContext, and the thread was still looking for ways to reduce the fragmentation. Rahila Syed and Ashutosh Bapat found little value in PlanCacheContext and SPICacheContext, which hold no allocations of their own. Michael Paquier marked the entry returned with feedback, citing the open feedback and concerns.
- **Reason (quote):** "Andres's concern, IIUC, is that the additional memory contexts will cause additional fragmentation." ([message](https://www.postgresql.org/message-id/4edadffe14ead8860f4d02141aaf154871fef0cb.camel%40j-davis.com))
- **What is different this time:** P1 added child contexts under CacheMemoryContext and stalled on the fragmentation they could add. This proposal also adds contexts, but on the new System context type, which has a small header and leaves allocation to malloc.


## 3. Objections a reviewer is likely to raise

### 1. Shared rd_supportinfo changes what support functions may cache in fn_extra

fixable · level 2, rubric item 7 · ABI / extension API, backpatchability · `replication-ondisk-1`

An extension index AM or opclass support function caches state in fn_extra, assuming it belongs to one index. After the patch, a second index with the same shape but a different collation or relation reuses that FmgrInfo. The function returns results built for the other index, or reads memory freed when the first index's relcache entry is dropped.

Until now each index had its own rd_supportinfo array allocated in its own rd_indexcxt, so fn_mcxt and fn_extra lived and died with one index. The shape key covers AM, key count, opclasses and opclass options, but not collation or the relation itself. Chao Li showed the fn_mcxt lifetime problem in message #4, and the author acknowledged it in message #9, but no later changelist says it was fixed. The proposal does not say which memory context owns a shared FmgrInfo or what extension authors may still put in fn_extra.

Thread: [#4 (Chao Li, 2026-09-01)](https://www.postgresql.org/message-id/D5FC8DF7-0A90-4417-AF1C-F68BEDA81FBC%40gmail.com), [#9 (Matthias van de Meent, 2026-09-01)](https://www.postgresql.org/message-id/CAEze2WgaftjGZjeoixEuHcysVN3-oet224XNwPC7EfE-CBKSAw%40mail.gmail.com), [#16 (Matthias van de Meent, 2026-09-08)](https://www.postgresql.org/message-id/CAEze2WgqYJBch-VNshqxx8umQ7YZJKS-9RToYujW%2BgX0qHJZ8g%40mail.gmail.com), [#25 (Matthias van de Meent, 2026-10-06)](https://www.postgresql.org/message-id/CAEze2WiaoHWrWQk%2BZBinfQPZz9yDTVaHbAwopdqxWChaEfQ66Q%40mail.gmail.com), [#27 (Matthias van de Meent, 2026-10-12)](https://www.postgresql.org/message-id/CAEze2Wh7X6dM2omJxzBuDC1tkG6yYtDs2m%3D603wV_%3D-_94-qcw%40mail.gmail.com), [#29 (Matthias van de Meent, 2026-10-14)](https://www.postgresql.org/message-id/CAEze2Wh1gOWFxWMnfUDPnyM9W9KDkV%2BTSnETERb9nu9mrxnsdw%40mail.gmail.com)

**Question:** Which context owns fn_mcxt and fn_extra for a shared rd_supportinfo in v5, and what does the patch promise extension support functions about per-index caching?

Counter-check: held. Message #9 accepts the lifetime problem, and no changelist from v2 to v5 says it is fixed or names the context that owns a shared FmgrInfo. The shape key in message #1 leaves out collation, so the fn_extra sharing question also stands.

### 2. No numbers for the extra cost of building index relcache entries

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A workload with many partitions runs frequent DDL or ANALYZE, and each run invalidates many index relcache entries. Every rebuild now does a shape hash lookup and every destroy does a deregistration, and SystemContext does one malloc per palloc. Users with plenty of memory see slower relcache rebuilds after upgrade and gain nothing.

The 0003 commit message admits extra work at load and calls it a worthy trade-off, but the thread has only memory numbers (messages #3, #22). A rebuild runs at the first access to each index in every backend and after every invalidation, so the cost falls on everyone. The memory savings help only sessions that touch many indexes.

Thread: [#3 (Matthias van de Meent, 2026-08-31)](https://www.postgresql.org/message-id/CAEze2WhMKEs4wxH3E%3Du_wJXdabdo0XzUHvOhSQ6hD8-mtjprRg%40mail.gmail.com), [#22 (David Geier, 2026-09-14)](https://www.postgresql.org/message-id/dfb1cada-e9c2-488c-80de-3f343ac74cbc%40googlemail.com)

**Question:** How much slower are backend start and relcache rebuild after mass invalidation, for example on the 148,000-index schema from message #22?

Counter-check: held. The 0003 commit message admits extra work when loading index relcache entries, and the thread has only memory numbers. Every backend pays this cost at first access and after each invalidation, so a timing measurement is a fair request.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | RelationData fields become const and shared, fmgr.h and c.h change, and extensions get a new System context API. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection replication-ondisk-1: Shared rd_supportinfo changes what support functions may cache in fn_extra |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new memory context type (systemctx.c) and a shape cache in relcache.c need care from now on. |
| performance on the hot path | **yes** | Every index relcache build and destroy adds a shape lookup, and System contexts do one malloc per palloc. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. In v5, which memory context owns fn_mcxt and fn_extra of a shared rd_supportinfo, and what keeps it alive until the last index using that shape is destroyed?
2. Can a support function still cache per-index or per-collation state in fn_extra, given that the shape key does not include rd_indcollation?
3. How much slower are backend start and relcache rebuild after mass invalidation, for example on the 148,000-index schema from message #22?
4. When are shape cache entries freed under heavy relcache invalidation?
5. How do fmgr.h and c.h change, and what do extension authors need to adjust?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. David Rowley reviewed the System context several times and sent a fixup; the deduplication patch still needs a committer who knows relcache invalidation and fmgr lifetime rules.
- **Liability:** The owner signs for memory lifetime of shared FmgrInfo under relcache invalidation, a new memory context type open to extensions, and const-ified RelationData fields, for as long as the relcache keeps this layout.
- **Author follow-up:** The author posted five versions over about two months, answered review quickly and kept the entry rebased.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users with many partitions or indexes gain per-backend memory, with a measured drop from 776 MiB to 633 MiB in message #22. The shared rd_supportinfo ownership rule should be settled before more review of the 42 KiB deduplication patch in relcache.c and fmgr.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Shared rd_supportinfo changes the fn_extra contract for opclass support functions. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same fn_extra sharing across indexes with different collations and also the lifetime problem from message #4.
- `standards-compat-2` System contexts hide per-allocation malloc overhead in memory views. Refuted (no failure scenario): pg_backend_memory_contexts has never reported malloc overhead or RSS, and a System context makes only a few allocations. Message #22 compares used_bytes, which leaves out free space in aset blocks on master, so it does not overstate the saving in the way the objection describes.
- `security-1` Shared rd_supportinfo lifetime bug lets any table owner crash the server. Refuted (not specific): Duplicate of replication-ondisk-1. The crash is the same dangling fn_mcxt problem from message #4, and the privilege framing adds no separate design issue.
- `maintenance-fit-1` Shared rd_supportinfo changes the fmgr cache lifetime contract. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same lifetime problem and the same missing rule for support functions that use fn_extra.
- `maintenance-fit-3` SystemContext savings depend on the platform malloc. Refuted (no failure scenario): Master already makes at least one malloc per "index info" context, so the patch adds no new dependence on the platform malloc. To cancel the drop from 2086 to 329 bytes per context, an allocator would have to waste far more than known size-class rounding does.

### Review angles

- prior-attempts: 0 objection(s). [P1] is the only related precedent, and this proposal targets its recorded concern about per-context overhead; Andres Freund, co-author of the System context, argues for more cache contexts in message #20. [P10] is about pruning catcache entries, a different idea. No earlier rejection reason applies unanswered, so I raise no objection from this angle.
- standards-compat: 2 objection(s). The proposal adds no SQL syntax, GUC, catalog object or dump-visible state. The remaining user-facing changes are the new context type and context names, plus const fields that fail loudly at compile time.
- replication-ondisk: 1 objection(s). The patch adds no on-disk, WAL, catalog or wire protocol state. All shape cache state is per-backend memory and is rebuilt on restart. The only cross-boundary concern is the extension API contract for shared FmgrInfo and the const fields.
- security: 1 objection(s). The proposal adds no SQL surface, no new privilege and no new monitoring data beyond context names and sizes in pg_backend_memory_contexts. The unbounded shape cache growth in message #24 was fixed in v3 (message #25), so it is not raised here.
- maintenance-fit: 3 objection(s). Committability is reasonable. Andres Freund co-wrote SystemContext and David Rowley reviewed it closely, but the deduplication patch has had less committer review.
- already-exists: 0 objection(s). No existing feature or extension meets this need, because relcache allocation of opclass-derived index fields can only be changed in core. An aset context with an allocChunkLimit of zero was the obvious alternative; message #3 shows its fixed overhead is too high (200 vs 108 bytes per context), and David Rowley and Andres Freund accepted the new System context type after that answer.
- demand: 0 objection(s). The material shows real demand. David Geier reports production workloads with 10-100k tables and hundreds of thousands of indexes (message #6). On one such schema he measured relcache memory falling from 776 MiB to 633 MiB (message #22). Partitioned tables with identical index definitions are a common case that benefits (messages #8, #10). Andres Freund co-wrote the System context and wants more cache contexts made cheap (message #20). The memory saving applies per backend. The open issues are the shared rd_supportinfo lifetime and the unmeasured load cost, and both belong to other angles.

### Precedents retrieved

- P1: [Separate memory contexts for relcache and catcache](https://commitfest.postgresql.org/patch/4554/) (cf, Returned with feedback, 2024-12-11) - used in the review; reason summarised from its thread
- P2: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P3: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P4: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P5: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P6: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P7: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant
- P8: [CONTEXT of ALTER TABLE behind-the-scenes effects](https://commitfest.postgresql.org/patch/427/) (cf, Returned with feedback, 2016-03-15) - judged not relevant; reason summarised from its thread
- P9: [Revert incorrect memory-conservation hack in inheritance_planner().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0f36045b2e3d037bb7647d84373404fa4ba9588) (revert, Reverted, 2011-01-13) - judged not relevant
- P10: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - used in the review; reason summarised from its thread
- P11: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P12: [Revert: Allow table AM to store complex data structures in rd_amcache](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=922c4c461d213a422ee7eb6c38e399607539210a) (revert, Reverted, 2024-04-11) - judged not relevant
- P13: [JIT: Optimize generated functions earlier to lower memory usage](https://commitfest.postgresql.org/patch/2298/) (cf, Withdrawn, 2019-11-12) - judged not relevant; reason summarised from its thread
- P14: [NOTIFY and pg_notify performance when deduplicating notifications](https://commitfest.postgresql.org/patch/1820/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P15: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P16: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P17: [Make psql \di show index with type info](https://commitfest.postgresql.org/patch/1083/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P18: [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (cf, Returned with feedback, 2017-03-07) - judged not relevant
- P19: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P20: [Opclass parameters](https://commitfest.postgresql.org/patch/1559/) (cf, Returned with feedback, 2018-03-01) - judged not relevant
- P21: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - judged not relevant
- P22: [Opclass parameters](https://commitfest.postgresql.org/patch/2183/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P23: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant
- P24: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0005-Relcache-Use-Proxy-context-for-index-info-context.patch not read
- attachment limit (8) reached; v1-0002-Move-allocations-of-relation-opclass-fields-to-In.patch not read
- attachment limit (8) reached; v1-0001-Mark-constant-opclass-related-fields-const-in-Rel.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 174 s; cost reported by the CLI: $3.05
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 85 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:31:32.938Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
