# Design pre-review: nbtree: opt-in leaf key prefix compression

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7523](https://commitfest.postgresql.org/patch/7523/); [thread](https://www.postgresql.org/message-id/flat/at-01F1wsXLRH0qo%40floki), 15 of 15 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support: other databases have prefix compression, and the cost falls on indexes that turn it on. The design is not yet agreed. The leaf format would be permanent while header size, packing and fixed-width keys are still open (standards-compat-1). The proposal gives no argument that a coded leaf can always be split (replication-ondisk-3), and redo reruns the layout routine (maintenance-fit-1). Peter Geoghegan has not replied since v9.

Not checked: Without the code, the split argument in _bt_prefix_split_needed, the completeness of the _bt_getitem() audit and the determinism of redo could not be checked. Only 3 of 15 messages were shown in full.

## Prior attempts

- [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://www.postgresql.org/message-id/flat/CAEze2Whwvr8aYcBf0BeBuPy8mJGtwxGvQYA9OGR5eLFh6Q_ZvA@mail.gmail.com) (related; Returned with feedback, 2022-02-07). "Evidently, this needs improvements in the (likely common) attcacheoff-enabled multi-column case; as I don't think we can reasonably commit a 10+% regression." ([message](https://www.postgresql.org/message-id/CAEze2WhyBT2bKZRdj_U0KS2Sbewa1XoO_BzgpzLC09sa5LUROg%40mail.gmail.com)) Different now: P2 skipped equal leading attributes during binary search and stored nothing new. This proposal changes the leaf format, so P2 is an analogy only for the regression risk on reader paths.
- [_bt_binsrch* improvements - equal-prefix-skip binary search](https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com) (related; Withdrawn, 2020-09-15). "Concurrent page deletions could occur, which could cause a scan that skips a prefix to miss that the page it landed on doesn't have the same common prefix anymore." ([message](https://www.postgresql.org/message-id/CAH2-WznPxocJKaz%3DXMqKqPauawi-vtB62oXT9SkHhDNb7cpV9Q%40mail.gmail.com)) Different now: P4 carried a known-equal prefix down the tree, which concurrent page deletion breaks. This proposal codes items only against anchors on the same leaf, so that failure does not apply.

## Objections a reviewer is likely to raise

### 1. On-disk format would be fixed while its basic layout is still in flux

Release 20 ships the v10 layout: 6-byte code header, MAXALIGNed items, byte-order sharing. Release 21 adopts unaligned packing or normalized keys for bigint and timestamptz. Every later release must then read both layouts in pg_upgraded indexes, with only page flags to tell them apart.

Message #9 says the compressed representation is supported forever. The format has already been replaced once, from v8 to v9. Message #15 still lists unaligned packing and a smaller header as open, and message #13 leaves normalized keys open. Message #14 offers a different zone-based layout, and message #15 says neither layout is plainly right.

**Question:** Which of the open format questions (header size, unaligned packing, normalized keys) will be settled before commit, and how would a later change be marked on disk?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#9](https://www.postgresql.org/message-id/CAH2-WznyrHYLaROHDtaZfYWn3u6hx7jpzsbpQO_vxk8jDFO8GA%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki), [#14](https://www.postgresql.org/message-id/CAAU0Mh1an_cU-MoTgwfagxn1Bf0v8qc%2BPD-2WnkUvVJKZOFPng%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki); `standards-compat-1`</sub>

### 2. No stated guarantee that a coded leaf can always be split

A user inserts a large key into a full coded leaf whose decoded contents are near the two-page limit. The retry path, which splits the existing items alone, still finds no legal split point. The insert then fails with an error on that page and keeps failing until the index is rebuilt, and the same layout may also be unreplayable on a standby.

Peter asked in message #9 for guarantees that a legal split is always available in pathological cases, and called it a property of the on-disk representation. The v10 commit message describes a cached decoded-size check plus an "existing items are split alone and the insert is retried" path, but it gives no argument for why the retry always succeeds. Message #3 also notes that the forced-image fallback is reachable only through an injection point. Peter has not replied since v9.

**Question:** What bound on decoded page contents, enforced on every insert, dedup and redo path, proves that both a split and its retry always find a legal point?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#3](https://www.postgresql.org/message-id/y8lIX15Mjse_ALeMMfsK1oakyoxjIU4p2HLHF8sQArBsYgFDvAZz7vVCW26vl6IP7AEMWQArSkqBxvyCa4OKdXe8eiZ6OxqgvIC0FnJEM7c%3D%40burd.me), [#9](https://www.postgresql.org/message-id/CAH2-WznyrHYLaROHDtaZfYWn3u6hx7jpzsbpQO_vxk8jDFO8GA%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki); `replication-ondisk-3`</sub>

### 3. Redo reruns the grouping heuristic, freezing it into the WAL format

A minor release changes how leaf groups are chosen, as v10 already did. The standby runs the new minor version and the primary the old one, the usual order during a minor upgrade. Redo lays out a different coded left half than the primary wrote, and with wal_consistency_checking off nobody notices until a later read or failover returns wrong results.

The v10 commit message says redo collects the left half's items as the primary did and lays them out with the same routine, so the grouping rule runs on both sides. Message #2 names this hazard: two copies that must agree exactly, or a standby diverges with no version bump to catch it. Message #15 changed that rule between v9 and v10. A shared routine removes the duplicated code, but every future fix to the heuristic becomes a WAL format change that is easy to miss in a backpatch.

**Question:** Why not log the chosen groups for both halves, so a later change to the grouping rule cannot make primary and standby build different pages?

<sub>fixable, level 2, WAL / replication / standby; messages [#2](https://www.postgresql.org/message-id/iMzeiKz_iFUwu8k8FTdo2CJ1Qp_x7SkvdLxEtYPhM3y_XfFY8mNM1fpSKoWimjR9ZKTz_4XT8BNCYKJWSq9w087DmY29cvnVrmvKCzLplnk%3D%40burd.me), [#15](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki); `maintenance-fit-1`</sub>

### 4. Every future leaf reader must use _bt_getitem, and a missed one stays silent

A later nbtree patch, or an extension that inspects btree pages, reads a leaf data item with PageGetItem as all existing code does. On plain pages it works and its tests pass. On a BTP_PREFIX page it reads a coded tuple as a real key, which shows up in the field as a wrong comparison or a wrong index-only scan result.

Message #1 says every read of a leaf data item must go through _bt_getitem, and the audit missed the array-scan look-ahead in _bt_readpage on its first pass. Old and new formats coexist in one index with no version bump, so this rule applies to all nbtree code from now on. The proposal has no check that fails loudly when PageGetItem returns a coded item. External tools that parse btree pages get no signal beyond a flag bit.

**Question:** What makes a leaf data item read that bypasses _bt_getitem fail loudly in assert builds, and what is the plan for out-of-tree code that reads btree pages?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/at-01F1wsXLRH0qo%40floki); `maintenance-fit-2`</sub>

### 5. No master-versus-patch numbers for the default-off path in v10

A user who never enables prefix_compression upgrades. Every leaf read now checks BTP_PREFIX and goes through _bt_getitem(), and _bt_compare() decodes on compressed pages. If that adds a few percent to descent or scans, every btree index pays for it, and users see it only as slower queries after the upgrade.

Message #1 showed the off path matching master, but that was for v1, which v9 and v10 replaced (message #13). The v10 commit message compares the index only with itself without the parameter, in the same binary. [P2] was returned because of about 2% insert and 15% reindex regressions on common multi-column indexes. Reader paths that every index uses carry the same risk here.

Precedent: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://www.postgresql.org/message-id/flat/CAEze2Whwvr8aYcBf0BeBuPy8mJGtwxGvQYA9OGR5eLFh6Q_ZvA@mail.gmail.com) (Returned with feedback, 2022-02-07)

**Question:** With prefix_compression off, how do v10 point lookups, range scans, inserts and CREATE INDEX compare with master on common single- and multi-column indexes?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/at-01F1wsXLRH0qo%40floki), [#13](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki), [#15](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki); `prior-attempts-1`</sub>

## Questions for the author

1. Which open format questions (header size, unaligned packing, normalized keys for fixed-width types) will be settled before commit, and how would a later layout change be marked on disk?
2. What bound on decoded page contents, kept by every insert, deduplication and redo path, proves that a split and its retry always find a legal split point?
3. Why does split redo rerun the layout routine for the left half instead of replaying logged groups and anchors for both halves?
4. With prefix_compression off, how does v10 compare with master on point lookups, range scans, inserts and CREATE INDEX for common single-column and multicolumn indexes?
5. Which real schemas, with their index definitions, have btree keys whose stored bytes share long prefixes, and how much do those indexes shrink with v10?

## Touch points

- **on-disk format / pg_upgrade:** New BTP_PREFIX leaf format with a prefix directory item coexists with plain pages in one index indefinitely.
- **WAL / replication / standby:** xl_btree_split gains a flags field, XLOG_PAGE_MAGIC is bumped, and redo rebuilds coded pages without full-page images.
- **ABI / extension API:** Out-of-tree code that reads leaf data items with PageGetItem gets coded bytes on BTP_PREFIX pages.
- **pg_dump / restore round-trip:** The new prefix_compression storage parameter must be dumped and restored like other btree reloptions.
- **backpatchability:** Redo reruns the layout routine, so a later fix to grouping changes what a standby on another minor version builds.
- **maintenance burden ("who maintains this in 10 years"):** A second permanent leaf format, and every future leaf reader must go through _bt_getitem().
- **performance on the hot path:** Every leaf read checks the page flag and goes through _bt_getitem(), also for indexes with the parameter off.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Geoghegan asked the design questions in message #9 but has not replied since v9. The owner must know nbtree page splits, deduplication and nbtree redo.
- Priority: fix design first; audience a sizeable minority; review cost high. Keys with long shared prefixes and low-cardinality leading columns are common enough to matter. The patch is about 175 KiB in one commit across nbtree insert, split, deduplication, search and redo, and the permanent format is still open.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
