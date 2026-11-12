# Design pre-review: Tepid: extend the heap-only tuple (HOT) update optimization model such that only those indexes directly impacted by the update need be updated

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6968](https://commitfest.postgresql.org/patch/6968/); [thread](https://www.postgresql.org/message-id/flat/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), 16 of 16 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is long-standing and no surviving objection is fatal, but the list has not agreed on the design. Fresh entries appear to keep stubs, and so pages, out of the visibility map while the row lives (replication-ondisk-1), and the feature is on for every table with no switch (demand-2). The bit 14 marker is not shown to survive GIN posting lists or to be safe for other table AMs (replication-ondisk-3, replication-ondisk-2). Message #1 itself asks for agreement on the format before freezing it.

Not checked: Only 3 of 16 messages and no code were read. Whether GIN keeps the marker, how prune treats fresh entries on a live row, and what VACUUM counts could not be checked.

## Prior attempts

- [Write Amplification Reduction Method (WARM)](https://www.postgresql.org/message-id/flat/CABOikdMNy6yowA+wTGK9RVd8iw+CzqHeQSGpW7Yka_4RSZ_LOQ@mail.gmail.com) (same idea; Returned with feedback, 2017-10-02). "Looks like I'll be short on bandwidth to pursue this further, given other work commitments including upcoming Postgres-XL 10 release." ([message](https://www.postgresql.org/message-id/CABOikdOm_R1NzYUqQQ85p04VxDJpvdp_V3feaKBY8ATMWTE%2BHw%40mail.gmail.com)) Different now: This proposal uses a per-hop attribute bitmap in place of WARM's value recheck and per-AM amrecheck. It has no one-update-per-chain limit and no convert-back step, and it handles bitmap scans with a TID offset marker.
- [partial heap only tuples](https://www.postgresql.org/message-id/flat/2ECBBCA0-4D8D-4841-8872-4A5BBDC063D2@amazon.com) (similar idea; Returned with feedback, 2021-11-04). "As no update has been posted, the patch still doesn't apply." ([message](https://www.postgresql.org/message-id/9417DBC8-0C3F-49FD-893D-1C833F8EA5F3%40yesql.se)) Different now: Like PHOT, fresh entries point at the new heap-only tuple. This proposal adds crossed-attribute staleness, stub collapse in place of restricted pruning, and handling for index create and drop over live chains.

## Objections a reviewer is likely to raise

### 1. Collapsed chains can keep pages out of the visibility map indefinitely

A table gets one SIU update on many rows and then becomes read-mostly. Fresh entries such as (11)->LP[2] stay live, so their stubs stay, and the pages are never set all-visible. Index-only scans keep fetching the heap, and every aggressive VACUUM rescans those pages until the rows are rewritten or the index is rebuilt.

Example 4a step 4 in message #1 reclaims stubs only "once every entry into the chain has been swept". While v4 lives, though, the fresh entries pointing at LP[2] and LP[3] are the only current entries for their indexes, and ambulkdelete removes only entries for LP_DEAD items. Commit 0006 keeps any page that holds a preserved stub non-all-visible, and message #15 concedes pages will be frozen less often. The proposal does not show that this state ends while the row is live.

**Question:** What sequence of operations returns a page holding a live SIU row to all-visible, without updating that row again or rebuilding its indexes?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#15](https://www.postgresql.org/message-id/84a886d1-37af-4da2-ad1b-15cea9c7e965%40app.fastmail.com); `replication-ondisk-1`</sub>

### 2. The stale marker must survive every index path that re-encodes TIDs

A GIN fresh entry is flushed from the pending list into a compressed posting list. If that encoding keeps only real offset bits, or reads the offset through the stripping accessor, the marker is lost. A BitmapAnd with a btree then returns zero rows for a matching row, the false negative from message #7 again.

Message #12 already found that nbtree deduplication copied flagged TIDs in a way the fix had not expected. GIN compresses item pointers into a fixed offset width sized for heap pages, and out-of-tree AMs may pack TIDs as well. Commit 0005 also calls the cost "self-healing" through collapse, but collapse rewrites only the heap. The flagged index tuple stays flagged until the entry is removed.

**Question:** Which in-core and out-of-tree storage paths re-encode heap TIDs, and how is it guaranteed that each one keeps bit 14 on disk?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#7](https://www.postgresql.org/message-id/CAPpHfdvOLrULXQn%2Bs%2BbEG4QiVrps%2B_3mg2EjUN39OYUKoBE20w%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/b078fb42-cecb-4868-8a65-11f9ce484868%40app.fastmail.com), [#12](https://www.postgresql.org/message-id/efe30c88-cf01-4c48-b76e-777547babcea%40app.fastmail.com); `replication-ondisk-3`</sub>

### 3. Every future consumer of heap items must handle stubs, with no loud failure

A later patch, or an extension, walks LP_NORMAL heap items or rewrites tuples the way CLUSTER does, and does not know about natts = 0 stubs or HEAP_INDEXED_UPDATED. It drops live rows or misreads the trailing bitmap. Nothing fails at build or test time, and users find out from missing query results.

Message #1 says every consumer of LP_NORMAL items must tolerate both new shapes. Messages #5, #10, #11 and #12 then report a series of silent failures found after posting: CLUSTER and VACUUM FULL losing rows, ordered GiST scans, pg_surgery, and amcheck false results. That pattern indicates future heap code will repeat it. WARM [P12] was set aside partly over on-disk format changes and the risk of data-corrupting bugs.

Precedent: [Write Amplification Reduction Method (WARM)](https://www.postgresql.org/message-id/flat/CABOikdMNy6yowA+wTGK9RVd8iw+CzqHeQSGpW7Yka_4RSZ_LOQ@mail.gmail.com) (Returned with feedback, 2017-10-02)

**Question:** What mechanism makes new or out-of-core code that reads heap items fail loudly, instead of silently, when it meets a stub or a HOT-indexed tuple?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#5](https://www.postgresql.org/message-id/a35235bb-a3b4-45cc-8e4a-7e57556a9250%40app.fastmail.com), [#10](https://www.postgresql.org/message-id/3dc5d57e-6c77-4713-b97e-bf0a5a891226%40app.fastmail.com), [#11](https://www.postgresql.org/message-id/ed69804a-aa09-489b-af3f-6c804b2efd94%40app.fastmail.com), [#12](https://www.postgresql.org/message-id/efe30c88-cf01-4c48-b76e-777547babcea%40app.fastmail.com); `maintenance-fit-2`</sub>

### 4. Global stripping of TID offset bit 14 affects every table AM

An out-of-core table AM numbers rows with synthetic offsets above 16383 and stores those TIDs in btree indexes. After upgrade, ItemPointerGetOffsetNumber and ItemPointerCompare silently clear bit 14. Index scans then fetch or order the wrong row, and the user finds out from wrong query results.

Commit 0005 changes the shared ItemPointer accessors for all TIDs and exempts only the SpecToken and MovedPartitions sentinels. The stated reason, MaxOffsetNumber, is a heap limit. Message #9 itself points out that other table AMs already use synthetic TIDs. Existing indexes of such AMs carry these offsets on disk across pg_upgrade.

**Question:** How does a table AM whose stored TIDs already use offset bit 14 keep correct index lookups and TID ordering after this change?

<sub>fixable, level 2, ABI / extension API; messages [#9](https://www.postgresql.org/message-id/b078fb42-cecb-4868-8a65-11f9ce484868%40app.fastmail.com); `replication-ondisk-2`</sub>

### 5. Costs fall on all tables while gains reach only multi-index partial updates

A reporting table that sometimes updates one indexed column starts taking HOT-indexed updates by default. Its pages stay non-all-visible while stubs remain, so index-only scans fetch from the heap, bitmap scans see lossy pages and freezing is delayed. The owner notices slower analytics and vacuum and has no switch to turn the feature off.

The feature is on for every non-catalog table, and message #16 asks for an opt-in table option with no answer in the material. Message #15 accepts that longer redirect chains and stub metadata will keep pages from being frozen. The 0005 commit message says any page with a live fresh entry becomes lossy for bitmap scans, and messages #1 and #3 show a 1 to 2% read-only loss, while heap growth, VACUUM and seqscan costs are not measured.

**Question:** What do heap-scan, index-only-scan and VACUUM workloads lose on tables that rarely benefit, and why should the feature not be opt-in per table?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#3](https://www.postgresql.org/message-id/57c9e23a-bb03-4d7c-81f5-551cd42f8b34%40app.fastmail.com), [#15](https://www.postgresql.org/message-id/84a886d1-37af-4da2-ad1b-15cea9c7e965%40app.fastmail.com), [#16](https://www.postgresql.org/message-id/8736715E-C8F7-4D47-95C6-3B02ED4F699A%40gmail.com); `demand-2`</sub>

## Questions for the author

1. What sequence of operations returns a page holding a live HOT-indexed row to all-visible, without updating that row again or rebuilding its indexes?
2. Why is the feature on for every non-catalog table, and what would a per-table option do to chains that already exist when it is turned off?
3. Which paths re-encode heap TIDs, such as GIN posting lists and nbtree deduplication, and how does each keep bit 14 on disk?
4. How do autovacuum thresholds, the index-vacuum bypass and INDEX_CLEANUP off account for stubs and stale leaves, and which test covers them?
5. Why can stale entries not be filtered inside index_getnext_slot, so that out-of-core callers stay correct without change?

## Touch points

- **on-disk format / pg_upgrade:** New infomask2 bit, trailing bitmap, natts = 0 stubs, and bit 14 in stored index TIDs.
- **WAL / replication / standby:** Prune WAL gains a stub sub-record, and logical apply is gated per subscription.
- **catalog changes:** New pg_subscription column, new pg_proc entry and new columns in the statistics views.
- **ABI / extension API:** Changes the table AM update contract, the ItemPointer accessors and what index_getnext_slot returns to callers.
- **SQL-standard conflict or non-standard syntax:** CREATE and ALTER SUBSCRIPTION gain the option hot_indexed_on_apply.
- **planner behaviour change:** Bitmap scans turn lossy and index-only scans heap-fetch on affected pages; CLUSTER changes its copy path.
- **pg_dump / restore round-trip:** pg_dump must emit the new subscription option.
- **psql UX conventions:** \dRs+ shows the new subscription option.
- **security / privileges:** pg_relation_hot_indexed_stats() scans a whole table and its execute privilege is not stated.
- **backpatchability:** Format bugs found after release need back-branch fixes with stubs and marked TIDs already on disk.
- **maintenance burden ("who maintains this in 10 years"):** Every consumer of LP_NORMAL heap items and of stored TIDs must handle the new shapes for ever.
- **performance on the hot path:** heap_update, the index scan chain walk and the shared ItemPointer accessors all change.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-2: Costs fall on all tables while gains reach only multi-index partial updates

## Who would commit it, and when

- Owner: qualified committer engaged. It needs someone who knows heap pruning, VACUUM, the visibility map and nbtree. Nathan Bossart co-authors the format and prune patches and Alexander Korotkov reviewed the bitmap scan path; nobody in the visible thread has offered to commit it.
- Priority: fix design first; audience a sizeable minority; review cost high. Tables with many secondary indexes gain up to 76% throughput in the author's runs, but the series changes the heap format, prune, VACUUM, tidbitmap.c and the index scan API. The open points on the visibility map, bit 14 and the opt-out should be settled before line-by-line review.

---

[roast-full.md](roast-full.md) has 7 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 12.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
