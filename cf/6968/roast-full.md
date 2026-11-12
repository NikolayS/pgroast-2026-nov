# Design pre-review: Tepid: extend the heap-only tuple (HOT) update optimization model such that only those indexes directly impacted by the update need be updated

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6968](https://commitfest.postgresql.org/patch/6968/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Tepid: selective index updates for heap relations](https://www.postgresql.org/message-id/flat/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), started 2026-06-30
- Messages read: 16 of 16
- Patch prose read: v63-0001-Add-tests-to-cover-a-variety-of-heap-HOT-update-.patch, v63-0002-Identify-modified-indexed-attributes-in-the-exec.patch, v63-0003-Add-the-HOT-indexed-on-disk-format-inline-attr-b.patch, v63-0004-Add-HOT-indexed-updates-selective-index-maintena.patch, v63-0005-Fix-a-BitmapAnd-BitmapOr-false-negative-on-HOT-i.patch, v63-0006-Collapse-dead-HOT-indexed-chains-to-xid-free-stu.patch, v63-0007-Teach-amcheck-to-recognize-HOT-indexed-chains-an.patch, v63-0008-Add-HOT-indexed-statistics-and-the-comprehensive.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is long-standing and no surviving objection is fatal, but the list has not agreed on the design. Fresh entries appear to keep stubs, and so pages, out of the visibility map while the row lives (replication-ondisk-1), and the feature is on for every table with no switch (demand-2). The bit 14 marker is not shown to survive GIN posting lists or to be safe for other table AMs (replication-ondisk-3, replication-ondisk-2). Message #1 itself asks for agreement on the format before freezing it.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 16 messages and no code were read. Whether GIN keeps the marker, how prune treats fresh entries on a live row, and what VACUUM counts could not be checked.

## The proposal, restated

**Problem.** An update that changes any indexed column today becomes a non-HOT update and inserts a new entry into every index, even indexes whose columns did not change. The author wants to cut the extra WAL, index bloat and VACUUM work this causes for tables with many secondary indexes.

**Proposed design.** The new version stays a heap-only tuple on the same page, carries an inline bitmap of the indexed attributes changed at that hop, and only the changed indexes get a fresh entry pointing at it. Readers union the bitmaps of hops crossed and drop entries whose index columns overlap; prune collapses dead prefixes into xid-free stubs. Fresh entries set bit 14 of the stored TID offset so bitmap scans treat that page as lossy.

**User-visible surface.**
- On-disk format: new infomask2 bit HEAP_INDEXED_UPDATED (0x0800), a trailing modified-attrs bitmap, and xid-free stub tuples with natts = 0.
- Index tuple TIDs: bit 14 of the offset marks a fresh entry as possibly stale.
- Subscription option hot_indexed_on_apply (off, subset_only, always), with a pg_subscription column and psql \dRs+ display.
- Statistics: pg_stat_all_tables.n_tup_hot_indexed_upd, per-index n_tup_hot_indexed_upd_matched and _skipped, and function pg_relation_hot_indexed_stats(regclass).
- pageinspect 1.14: bt_page_items gains a hot_indexed column and reports ctid/htid with the marker removed.

**What the author claims.**
- Throughput gains up to 76% and WAL per update cuts up to 59% on a 128-vCPU host when few of 17 indexes change.
- Read-only index scans and the classic HOT control lose about 1 to 2%.
- The bitmap avoids the duplicate rows a value recheck returns after an ABA cycle, and no index AM needs changes.
- pg_upgrade is unaffected, since older clusters carry no such tuples.
- It answers the WARM objections, with no per-chain update limit and no convert-back step.

**Thread state.** Korotkov found a BitmapAnd false negative, which the author answered with the TID bit 14 marker, later fixes covered CLUSTER row loss and amcheck. Rupireddy asked to split the series, questioned moving modified-attribute detection into the executor, and asked about trade-offs; Chao Li asked for an opt-in table option and questioned the scan API change.

## 2. Prior attempts

### Write Amplification Reduction Method (WARM) (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/775/](https://commitfest.postgresql.org/patch/775/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CABOikdMNy6yowA+wTGK9RVd8iw+CzqHeQSGpW7Yka_4RSZ_LOQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CABOikdMNy6yowA+wTGK9RVd8iw+CzqHeQSGpW7Yka_4RSZ_LOQ@mail.gmail.com)
- **Date:** decided 2017-10-02; started 2016-08-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Robert Haas, Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** In April 2017 the patch was moved out of the release cycle. Reviewers, as quoted in the author's replies, were concerned about on-disk format changes, the risk of data-corrupting bugs, and performance regressions found in testing (for example, recheck overhead) that they considered not fully addressed. In July 2017 the author said he no longer had time to pursue it, and Peter Geoghegan questioned whether WARM helps the worst case, such as long-running transactions. The CF manager returned it with feedback because more work was needed and the patch had not moved in the current commitfest.
- **Reason (quote):** "Looks like I'll be short on bandwidth to pursue this further, given other work commitments including upcoming Postgres-XL 10 release." ([message](https://www.postgresql.org/message-id/CABOikdOm_R1NzYUqQQ85p04VxDJpvdp_V3feaKBY8ATMWTE%2BHw%40mail.gmail.com))
- **What is different this time:** This proposal uses a per-hop attribute bitmap in place of WARM's value recheck and per-AM amrecheck. It has no one-update-per-chain limit and no convert-back step, and it handles bitmap scans with a TID offset marker.

### partial heap only tuples (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2981/](https://commitfest.postgresql.org/patch/2981/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2ECBBCA0-4D8D-4841-8872-4A5BBDC063D2@amazon.com](https://www.postgresql.org/message-id/flat/2ECBBCA0-4D8D-4841-8872-4A5BBDC063D2@amazon.com)
- **Date:** decided 2021-11-04; started 2021-02-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch stopped applying to HEAD, and the entry was set to Waiting for Author in July 2021. No updated patch was posted, so the entry was returned with feedback in November 2021. The visible messages show design concerns, such as index create/drop handling and long-term heap fragmentation, but no objection that decided the outcome.
- **Reason (quote):** "As no update has been posted, the patch still doesn't apply." ([message](https://www.postgresql.org/message-id/9417DBC8-0C3F-49FD-893D-1C833F8EA5F3%40yesql.se))
- **What is different this time:** Like PHOT, fresh entries point at the new heap-only tuple. This proposal adds crossed-attribute staleness, stub collapse in place of restricted pruning, and handling for index create and drop over live chains.


## 3. Objections a reviewer is likely to raise

### 1. Collapsed chains can keep pages out of the visibility map indefinitely

fixable · level 2, rubric item 14 · on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A table gets one SIU update on many rows and then becomes read-mostly. Fresh entries such as (11)->LP[2] stay live, so their stubs stay, and the pages are never set all-visible. Index-only scans keep fetching the heap, and every aggressive VACUUM rescans those pages until the rows are rewritten or the index is rebuilt.

Example 4a step 4 in message #1 reclaims stubs only "once every entry into the chain has been swept". While v4 lives, though, the fresh entries pointing at LP[2] and LP[3] are the only current entries for their indexes, and ambulkdelete removes only entries for LP_DEAD items. Commit 0006 keeps any page that holds a preserved stub non-all-visible, and message #15 concedes pages will be frozen less often. The proposal does not show that this state ends while the row is live.

Thread: [#1 ("Greg Burd", 2026-06-30)](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#15 ("Greg Burd", 2026-07-20)](https://www.postgresql.org/message-id/84a886d1-37af-4da2-ad1b-15cea9c7e965%40app.fastmail.com)

**Question:** What sequence of operations returns a page holding a live SIU row to all-visible, without updating that row again or rebuilding its indexes?

Counter-check: held. Example 4a step 4 frees LP[2] and LP[3] while v4 is live, but the fresh entries (11)->LP[2] and (21)->LP[3] are the only current entries for their indexes. The material does not say what removes or re-points them.

### 2. The stale marker must survive every index path that re-encodes TIDs

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, ABI / extension API · `replication-ondisk-3`

A GIN fresh entry is flushed from the pending list into a compressed posting list. If that encoding keeps only real offset bits, or reads the offset through the stripping accessor, the marker is lost. A BitmapAnd with a btree then returns zero rows for a matching row, the false negative from message #7 again.

Message #12 already found that nbtree deduplication copied flagged TIDs in a way the fix had not expected. GIN compresses item pointers into a fixed offset width sized for heap pages, and out-of-tree AMs may pack TIDs as well. Commit 0005 also calls the cost "self-healing" through collapse, but collapse rewrites only the heap. The flagged index tuple stays flagged until the entry is removed.

Thread: [#7 (Alexander Korotkov, 2026-07-09)](https://www.postgresql.org/message-id/CAPpHfdvOLrULXQn%2Bs%2BbEG4QiVrps%2B_3mg2EjUN39OYUKoBE20w%40mail.gmail.com), [#9 ("Greg Burd", 2026-07-11)](https://www.postgresql.org/message-id/b078fb42-cecb-4868-8a65-11f9ce484868%40app.fastmail.com), [#12 ("Greg Burd", 2026-07-13)](https://www.postgresql.org/message-id/efe30c88-cf01-4c48-b76e-777547babcea%40app.fastmail.com)

**Question:** Which in-core and out-of-tree storage paths re-encode heap TIDs, and how is it guaranteed that each one keeps bit 14 on disk?

Counter-check: held. GIN posting lists pack heap offsets into 11 bits, and commit 0005 touches no GIN file, so the claim that GIN needs no change is not shown for compressed lists. Message #12 shows nbtree deduplication already caused one such surprise.

### 3. Every future consumer of heap items must handle stubs, with no loud failure

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), on-disk format / pg_upgrade, ABI / extension API · `maintenance-fit-2`

A later patch, or an extension, walks LP_NORMAL heap items or rewrites tuples the way CLUSTER does, and does not know about natts = 0 stubs or HEAP_INDEXED_UPDATED. It drops live rows or misreads the trailing bitmap. Nothing fails at build or test time, and users find out from missing query results.

Message #1 says every consumer of LP_NORMAL items must tolerate both new shapes. Messages #5, #10, #11 and #12 then report a series of silent failures found after posting: CLUSTER and VACUUM FULL losing rows, ordered GiST scans, pg_surgery, and amcheck false results. That pattern indicates future heap code will repeat it. WARM [P12] was set aside partly over on-disk format changes and the risk of data-corrupting bugs.

**Precedents.**
- [Write Amplification Reduction Method (WARM)](https://commitfest.postgresql.org/patch/775/) (Returned with feedback, 2017-10-02); thread: [link](https://www.postgresql.org/message-id/flat/CABOikdMNy6yowA+wTGK9RVd8iw+CzqHeQSGpW7Yka_4RSZ_LOQ@mail.gmail.com)

Thread: [#1 ("Greg Burd", 2026-06-30)](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#5 ("Greg Burd", 2026-07-06)](https://www.postgresql.org/message-id/a35235bb-a3b4-45cc-8e4a-7e57556a9250%40app.fastmail.com), [#10 ("Greg Burd", 2026-07-11)](https://www.postgresql.org/message-id/3dc5d57e-6c77-4713-b97e-bf0a5a891226%40app.fastmail.com), [#11 ("Greg Burd", 2026-07-13)](https://www.postgresql.org/message-id/ed69804a-aa09-489b-af3f-6c804b2efd94%40app.fastmail.com), [#12 ("Greg Burd", 2026-07-13)](https://www.postgresql.org/message-id/efe30c88-cf01-4c48-b76e-777547babcea%40app.fastmail.com)

**Question:** What mechanism makes new or out-of-core code that reads heap items fail loudly, instead of silently, when it meets a stub or a HOT-indexed tuple?

Counter-check: held. Message #1 says every consumer of LP_NORMAL items must tolerate both shapes, and messages #5 to #12 report several silent misses. A later version could add a loud check or a distinct item state, so this is fixable.

### 4. Global stripping of TID offset bit 14 affects every table AM

fixable · level 2, rubric item 11 · ABI / extension API, on-disk format / pg_upgrade · `replication-ondisk-2`

An out-of-core table AM numbers rows with synthetic offsets above 16383 and stores those TIDs in btree indexes. After upgrade, ItemPointerGetOffsetNumber and ItemPointerCompare silently clear bit 14. Index scans then fetch or order the wrong row, and the user finds out from wrong query results.

Commit 0005 changes the shared ItemPointer accessors for all TIDs and exempts only the SpecToken and MovedPartitions sentinels. The stated reason, MaxOffsetNumber, is a heap limit. Message #9 itself points out that other table AMs already use synthetic TIDs. Existing indexes of such AMs carry these offsets on disk across pg_upgrade.

Thread: [#9 ("Greg Burd", 2026-07-11)](https://www.postgresql.org/message-id/b078fb42-cecb-4868-8a65-11f9ce484868%40app.fastmail.com)

**Question:** How does a table AM whose stored TIDs already use offset bit 14 keep correct index lookups and TID ordering after this change?

Counter-check: held. Commit 0005 changes the shared ItemPointer accessors for all relations, and its justification, MaxOffsetNumber, is a page-layout limit that synthetic TIDs need not obey. The scenario names no existing AM, but message #15 shows the author is also unsure about bit 14.

### 5. Costs fall on all tables while gains reach only multi-index partial updates

fixable · level 1b, rubric item 4 · desirability, performance on the hot path, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

A reporting table that sometimes updates one indexed column starts taking HOT-indexed updates by default. Its pages stay non-all-visible while stubs remain, so index-only scans fetch from the heap, bitmap scans see lossy pages and freezing is delayed. The owner notices slower analytics and vacuum and has no switch to turn the feature off.

The feature is on for every non-catalog table, and message #16 asks for an opt-in table option with no answer in the material. Message #15 accepts that longer redirect chains and stub metadata will keep pages from being frozen. The 0005 commit message says any page with a live fresh entry becomes lossy for bitmap scans, and messages #1 and #3 show a 1 to 2% read-only loss, while heap growth, VACUUM and seqscan costs are not measured.

Thread: [#1 ("Greg Burd", 2026-06-30)](https://www.postgresql.org/message-id/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com), [#3 ("Greg Burd", 2026-07-02)](https://www.postgresql.org/message-id/57c9e23a-bb03-4d7c-81f5-551cd42f8b34%40app.fastmail.com), [#15 ("Greg Burd", 2026-07-20)](https://www.postgresql.org/message-id/84a886d1-37af-4da2-ad1b-15cea9c7e965%40app.fastmail.com), [#16 (Chao Li, 2026-07-28)](https://www.postgresql.org/message-id/8736715E-C8F7-4D47-95C6-3B02ED4F699A%40gmail.com)

**Question:** What do heap-scan, index-only-scan and VACUUM workloads lose on tables that rarely benefit, and why should the feature not be opt-in per table?

Counter-check: held. The feature is on for all non-catalog tables, commit 0005 accepts lossy bitmap pages, and message #15 concedes delayed freezing. The request for an opt-in table option in message #16 has no reply in the visible thread.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New infomask2 bit, trailing bitmap, natts = 0 stubs, and bit 14 in stored index TIDs. |
| WAL / replication / standby | **yes** | Prune WAL gains a stub sub-record, and logical apply is gated per subscription. |
| wire protocol | no |  |
| catalog changes | **yes** | New pg_subscription column, new pg_proc entry and new columns in the statistics views. |
| ABI / extension API | **yes** | Changes the table AM update contract, the ItemPointer accessors and what index_getnext_slot returns to callers. |
| SQL-standard conflict or non-standard syntax | **yes** | CREATE and ALTER SUBSCRIPTION gain the option hot_indexed_on_apply. |
| new GUC | no |  |
| planner behaviour change | **yes** | Bitmap scans turn lossy and index-only scans heap-fetch on affected pages; CLUSTER changes its copy path. |
| pg_dump / restore round-trip | **yes** | pg_dump must emit the new subscription option. |
| psql UX conventions | **yes** | \dRs+ shows the new subscription option. |
| security / privileges | **yes** | pg_relation_hot_indexed_stats() scans a whole table and its execute privilege is not stated. |
| backpatchability | **yes** | Format bugs found after release need back-branch fixes with stubs and marked TIDs already on disk. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every consumer of LP_NORMAL heap items and of stored TIDs must handle the new shapes for ever. |
| performance on the hot path | **yes** | heap_update, the index scan chain walk and the shared ItemPointer accessors all change. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-2: Costs fall on all tables while gains reach only multi-index partial updates |

## 5. Questions the author must answer before anyone reviews code

1. What sequence of operations returns a page holding a live HOT-indexed row to all-visible, without updating that row again or rebuilding its indexes?
2. Why is the feature on for every non-catalog table, and what would a per-table option do to chains that already exist when it is turned off?
3. Which paths re-encode heap TIDs, such as GIN posting lists and nbtree deduplication, and how does each keep bit 14 on disk?
4. How do autovacuum thresholds, the index-vacuum bypass and INDEX_CLEANUP off account for stubs and stale leaves, and which test covers them?
5. Why can stale entries not be filtered inside index_getnext_slot, so that out-of-core callers stay correct without change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. It needs someone who knows heap pruning, VACUUM, the visibility map and nbtree. Nathan Bossart co-authors the format and prune patches and Alexander Korotkov reviewed the bitmap scan path; nobody in the visible thread has offered to commit it.
- **Liability:** A permanent heap tuple format, a marker bit in every stored index TID, and correctness of every scan, prune and rewrite path over them, with no way to remove the format later.
- **Author follow-up:** Strong sign: the author posted versions v48 to v63 with rebases, bug fixes and a second benchmark run within the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Tables with many secondary indexes gain up to 76% throughput in the author's runs, but the series changes the heap format, prune, VACUUM, tidbitmap.c and the index scan API. The open points on the visibility map, bit 14 and the opt-out should be settled before line-by-line review.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Long-snapshot worst case raised against WARM is not measured here. Not listed: not among the strongest objections.
- `standards-compat-1` (fixable, level 2) CLUSTER USING a GiST index silently stops clustering on affected tables. Not listed: not among the strongest objections.
- `standards-compat-3` (fixable, level 2) hot_indexed_on_apply exposes a storage internal as a subscription option. Not listed: not among the strongest objections.
- `standards-compat-4` (fixable, level 2) Out-of-core index scan callers must learn a new stale flag. Not listed: not among the strongest objections.
- `replication-ondisk-4` (fixable, level 2) Nothing tells VACUUM that stubs and stale leaves still need a pass. Not listed: not among the strongest objections.
- `security-1` (fixable, level 2) Default execute privilege for pg_relation_hot_indexed_stats() is not stated. Not listed: not among the strongest objections.
- `demand-1` (fixable, level 1b) No field evidence on how often workloads change only some indexed columns. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-2` Correctness risk that stalled WARM recurs, and the feature cannot be turned off. Refuted (not specific): Duplicate of demand-2 for the missing per-table switch. The rest infers future bugs from bugs already fixed and names no scan path that is still wrong.
- `standards-compat-2` Feature is on for every table with no per-table opt-out. Refuted (not specific): Duplicate of demand-2, which makes the same request for a per-table opt-in with the costs listed more fully.
- `maintenance-fit-1` Stubs and stale leaves can outlive VACUUM when index cleanup is skipped. Refuted (not specific): Duplicate of replication-ondisk-4, which raises the same VACUUM bookkeeping gap and states the test coverage more carefully.
- `maintenance-fit-3` Index scan callers outside core must learn a new stale flag. Refuted (not specific): Duplicate of standards-compat-4 for the stale flag, and of replication-ondisk-2 for the bit 14 part.
- `maintenance-fit-4` Always on, with no per-table switch for workloads that lose. Refuted (not specific): Duplicate of demand-2, which cites the same messages and asks the same question.

### Review angles

- prior-attempts: 2 objection(s). P1 (BRIN HOT revert) and P14 (local_update_limit) share the area, but their recorded reasons do not apply to this design.
- standards-compat: 4 objection(s). Objections were found from the standards and compatibility angle.
- replication-ondisk: 4 objection(s). Objections were found, so no note is needed. WAL replay on standbys and logical apply gating were reviewed, and the material addresses them well enough for this angle.
- security: 1 objection(s). The core heap and index design does not change who can see which rows, because the staleness test only filters entries. pageinspect and pg_surgery stay superuser tools, and the new counters fit existing pg_stat views.
- maintenance-fit: 4 objection(s). Objections were found.
- already-exists: 0 objection(s). No existing Postgres feature, setting or documented idiom skips maintenance of unchanged indexes when an indexed column changes. fillfactor and dropping indexes do not do this, and the closest workaround, a separate table access method, is a different storage engine rather than a modest-effort alternative. The feature changes the heap tuple format, heap_update, prune and tidbitmap.c, so no extension or hook can deliver it outside core. Earlier attempts at the same idea ([P12], [P3]) are prior designs, not existing features, and other angles cover them.
- demand: 2 objection(s). Write amplification from non-HOT updates is a long-standing problem, and the material shows lasting interest across several designs. The objections ask for the size of the group helped and for the cost to non-users, not for proof that the problem exists.

### Precedents retrieved

- P1: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - used in the review
- P2: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P3: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - used in the review; reason summarised from its thread
- P4: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P5: [Revert "Prevent incorrect updates of pg_index while reindexing pg_index itself."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c096d19b74a637443109e528000342e896b150f3) (revert, Reverted, 2011-04-19) - judged not relevant
- P6: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P7: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P8: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P9: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P10: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P11: [On-disk bitmap indexes](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P12: [Write Amplification Reduction Method (WARM)](https://commitfest.postgresql.org/patch/775/) (cf, Returned with feedback, 2017-10-02) - used in the review; reason summarised from its thread
- P13: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P14: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - used in the review; reason summarised from its thread
- P15: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant; reason summarised from its thread
- P16: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P17: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P18: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P19: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P20: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P21: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P22: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P23: [Fix the statistics gathering in case of commit and chain](https://commitfest.postgresql.org/patch/3657/) (cf, Returned with feedback, 2022-10-06) - judged not relevant
- P24: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/af02b486-6adc-494c-a357-1dd72f655dcf%40app.fastmail.com). Others: https://www.postgresql.org/message-id/flat/3604ec98-147d-4ba5-b215-412fa2797633%40app.fastmail.com
- attachment limit (8) reached; v63-0009-Gate-HOT-indexed-updates-on-the-logical-replicat.patch not read
- attachment limit (8) reached; v63-0010-DO-NOT-MERGE-Add-a-HOT-SIU-benchmark-harness.patch not read
- attachment limit (8) reached; v48-0001-Add-tests-to-cover-a-variety-of-heap-HOT-update-.patch not read
- attachment limit (8) reached; v48-0002-Identify-modified-indexed-attributes-in-the-exec.patch not read
- attachment limit (8) reached; v48-0003-Add-the-HOT-indexed-on-disk-format-inline-attr-b.patch not read
- attachment limit (8) reached; v48-0004-Add-HOT-indexed-updates-selective-index-maintena.patch not read
- attachment limit (8) reached; v48-0005-Collapse-dead-HOT-indexed-chains-to-xid-free-stu.patch not read
- attachment limit (8) reached; v48-0006-Teach-amcheck-to-recognize-HOT-indexed-chains-an.patch not read
- attachment limit (8) reached; v48-0007-Add-HOT-indexed-statistics-and-the-comprehensive.patch not read
- attachment limit (8) reached; v48-0008-Gate-HOT-indexed-updates-on-the-logical-replicat.patch not read
- attachment limit (8) reached; v48-0009-DO-NOT-MERGE-Add-a-HOT-SIU-benchmark-harness.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 273 s; cost reported by the CLI: $6.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 144 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:11:30.428Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
