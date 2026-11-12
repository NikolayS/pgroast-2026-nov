# Design pre-review: Reducing relcache memory usage: deduping index shapes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7237](https://commitfest.postgresql.org/patch/7237/); [thread](https://www.postgresql.org/message-id/flat/CAEze2WiPyruOtUOSyRUV8mQssjmYwno0M6hkxC_iUpH-%3DW8WcA%40mail.gmail.com), 31 of 31 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal has shown demand and no objection survived at levels 1a or 1b. Two fixable level 2 points remain. replication-ondisk-1: message #9 accepts the dangling fn_mcxt problem in shared rd_supportinfo, and no changelist through v5 says it is fixed. maintenance-fit-2: the 0003 commit message admits extra load work, and the thread has only memory numbers. Both need an answer from the author and a committer who knows relcache and fmgr.

Not checked: Without the code, it is not possible to check whether v5 already fixes the fn_mcxt lifetime, how the shape cache frees entries, or what the fmgr.h and c.h changes do.

## Prior attempts

- [Separate memory contexts for relcache and catcache](https://www.postgresql.org/message-id/flat/CAGPVpCTJWEQLt2eOSDGTDtRbQPUQ9b9JtZWro9osJubTyWAEMA@mail.gmail.com) (related; Returned with feedback, 2024-12-11). "Andres's concern, IIUC, is that the additional memory contexts will cause additional fragmentation." ([message](https://www.postgresql.org/message-id/4edadffe14ead8860f4d02141aaf154871fef0cb.camel%40j-davis.com)) Different now: P1 added child contexts under CacheMemoryContext and stalled on the fragmentation they could add. This proposal also adds contexts, but on the new System context type, which has a small header and leaves allocation to malloc.

## Objections a reviewer is likely to raise

### 1. Shared rd_supportinfo changes what support functions may cache in fn_extra

An extension index AM or opclass support function caches state in fn_extra, assuming it belongs to one index. After the patch, a second index with the same shape but a different collation or relation reuses that FmgrInfo. The function returns results built for the other index, or reads memory freed when the first index's relcache entry is dropped.

Until now each index had its own rd_supportinfo array allocated in its own rd_indexcxt, so fn_mcxt and fn_extra lived and died with one index. The shape key covers AM, key count, opclasses and opclass options, but not collation or the relation itself. Chao Li showed the fn_mcxt lifetime problem in message #4, and the author acknowledged it in message #9, but no later changelist says it was fixed. The proposal does not say which memory context owns a shared FmgrInfo or what extension authors may still put in fn_extra.

**Question:** Which context owns fn_mcxt and fn_extra for a shared rd_supportinfo in v5, and what does the patch promise extension support functions about per-index caching?

<sub>fixable, level 2, ABI / extension API; messages [#4](https://www.postgresql.org/message-id/D5FC8DF7-0A90-4417-AF1C-F68BEDA81FBC%40gmail.com), [#9](https://www.postgresql.org/message-id/CAEze2WgaftjGZjeoixEuHcysVN3-oet224XNwPC7EfE-CBKSAw%40mail.gmail.com), [#16](https://www.postgresql.org/message-id/CAEze2WgqYJBch-VNshqxx8umQ7YZJKS-9RToYujW%2BgX0qHJZ8g%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAEze2WiaoHWrWQk%2BZBinfQPZz9yDTVaHbAwopdqxWChaEfQ66Q%40mail.gmail.com), [#27](https://www.postgresql.org/message-id/CAEze2Wh7X6dM2omJxzBuDC1tkG6yYtDs2m%3D603wV_%3D-_94-qcw%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAEze2Wh1gOWFxWMnfUDPnyM9W9KDkV%2BTSnETERb9nu9mrxnsdw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. No numbers for the extra cost of building index relcache entries

A workload with many partitions runs frequent DDL or ANALYZE, and each run invalidates many index relcache entries. Every rebuild now does a shape hash lookup and every destroy does a deregistration, and SystemContext does one malloc per palloc. Users with plenty of memory see slower relcache rebuilds after upgrade and gain nothing.

The 0003 commit message admits extra work at load and calls it a worthy trade-off, but the thread has only memory numbers (messages #3, #22). A rebuild runs at the first access to each index in every backend and after every invalidation, so the cost falls on everyone. The memory savings help only sessions that touch many indexes.

**Question:** How much slower are backend start and relcache rebuild after mass invalidation, for example on the 148,000-index schema from message #22?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/CAEze2WhMKEs4wxH3E%3Du_wJXdabdo0XzUHvOhSQ6hD8-mtjprRg%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/dfb1cada-e9c2-488c-80de-3f343ac74cbc%40googlemail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. In v5, which memory context owns fn_mcxt and fn_extra of a shared rd_supportinfo, and what keeps it alive until the last index using that shape is destroyed?
2. Can a support function still cache per-index or per-collation state in fn_extra, given that the shape key does not include rd_indcollation?
3. How much slower are backend start and relcache rebuild after mass invalidation, for example on the 148,000-index schema from message #22?
4. When are shape cache entries freed under heavy relcache invalidation?
5. How do fmgr.h and c.h change, and what do extension authors need to adjust?

## Touch points

- **ABI / extension API:** RelationData fields become const and shared, fmgr.h and c.h change, and extensions get a new System context API.
- **backpatchability:** see objection replication-ondisk-1: Shared rd_supportinfo changes what support functions may cache in fn_extra
- **maintenance burden ("who maintains this in 10 years"):** A new memory context type (systemctx.c) and a shape cache in relcache.c need care from now on.
- **performance on the hot path:** Every index relcache build and destroy adds a shape lookup, and System contexts do one malloc per palloc.

## Who would commit it, and when

- Owner: qualified committer engaged. David Rowley reviewed the System context several times and sent a fixup; the deduplication patch still needs a committer who knows relcache invalidation and fmgr lifetime rules.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users with many partitions or indexes gain per-backend memory, with a measured drop from 776 MiB to 633 MiB in message #22. The shared rd_supportinfo ownership rule should be settled before more review of the 42 KiB deduplication patch in relcache.c and fmgr.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 3.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
