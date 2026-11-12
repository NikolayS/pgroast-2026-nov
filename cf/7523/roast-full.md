# Design pre-review: nbtree: opt-in leaf key prefix compression

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7523](https://commitfest.postgresql.org/patch/7523/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [nbtree: opt-in leaf key prefix compression](https://www.postgresql.org/message-id/flat/at-01F1wsXLRH0qo%40floki), started 2026-10-26
- Messages read: 15 of 15
- Patch prose read: v10-0001-nbtree-opt-in-leaf-key-prefix-compression.patch, 0001-nbtree-opt-in-leaf-key-prefix-compression.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support: other databases have prefix compression, and the cost falls on indexes that turn it on. The design is not yet agreed. The leaf format would be permanent while header size, packing and fixed-width keys are still open (standards-compat-1). The proposal gives no argument that a coded leaf can always be split (replication-ondisk-3), and redo reruns the layout routine (maintenance-fit-1). Peter Geoghegan has not replied since v9.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the split argument in _bt_prefix_split_needed, the completeness of the _bt_getitem() audit and the determinism of redo could not be checked. Only 3 of 15 messages were shown in full.

## The proposal, restated

**Problem.** Btree indexes on keys with long shared leading bytes store that prefix again in every tuple, so they are larger than needed and cost more I/O and WAL. The author names URLs, reverse-DNS names, file paths, tenant-prefixed keys and many btree indexes on jsonb paths, common in apps that moved from MongoDB.

**Proposed design.** With prefix_compression on, a leaf written whole (build or split half) gets a BTP_PREFIX flag and a prefix directory item holding one or more groups, each with an anchor. Each data item stores only the bytes after its shared prefix, compared by stored bytes, not opclass equality. Old and new pages coexist with no BTREE_VERSION change, and redo rebuilds coded pages without full-page images.

**User-visible surface.**
- New btree storage parameter prefix_compression (boolean, default off), settable with ALTER INDEX without rewriting the index.
- New page flag BTP_PREFIX (bit 512 of btpo_flags) and a prefix directory item on compressed leaf pages.
- pageinspect bt_page_items shows coded items decoded, with itemlen giving the stored size.
- xl_btree_split gains a flags field, which needs an XLOG_PAGE_MAGIC bump.
- amcheck checks the directory and its cached decoded sizes.

**What the author claims.**
- On 2 million URL-like keys the index is 35-37% smaller and inserts write 17% less WAL, but inserts and CREATE INDEX take about 10% longer.
- Warm reads are unchanged and cold-cache index scans are 25-35% faster.
- Prefix compression composes with deduplication and bottom-up deletion on one page.
- Byte-equality sharing is correct for every type, including user-defined ones, with no equalimage support function.
- pg_upgrade, standbys and existing indexes need no REINDEX.

**Thread state.** Peter Geoghegan asked for the motivation, the design trade-offs, and split guarantees in pathological cases before any WAL detail, and posted his own earlier prototype. Greg replaced v1-v8 with a multi-column, multi-prefix design (v9, v10) and asks for agreement on it; Yao Feng reviewed grouping and the per-item header cost.

## 2. Prior attempts

### Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation (related)

- **Link:** [https://commitfest.postgresql.org/patch/3078/](https://commitfest.postgresql.org/patch/3078/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAEze2Whwvr8aYcBf0BeBuPy8mJGtwxGvQYA9OGR5eLFh6Q_ZvA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAEze2Whwvr8aYcBf0BeBuPy8mJGtwxGvQYA9OGR5eLFh6Q_ZvA@mail.gmail.com)
- **Date:** decided 2022-02-07; started 2021-04-15
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author's own June 2021 benchmarks showed regressions in multi-column indexes that can use attcacheoff: about 2% for insert and 15% for reindex with both patches. The author said this had to be fixed before commit, but no fixed version appears in the thread. In December 2021 Daniel Gustafsson moved the entry forward and noted that the patch no longer applied and needed a rebase. The visible messages do not show a rebased version or the final decision.
- **Reason (quote):** "Evidently, this needs improvements in the (likely common) attcacheoff-enabled multi-column case; as I don't think we can reasonably commit a 10+% regression." ([message](https://www.postgresql.org/message-id/CAEze2WhyBT2bKZRdj_U0KS2Sbewa1XoO_BzgpzLC09sa5LUROg%40mail.gmail.com))
- **What is different this time:** P2 skipped equal leading attributes during binary search and stored nothing new. This proposal changes the leaf format, so P2 is an analogy only for the regression risk on reader paths.

### _bt_binsrch* improvements - equal-prefix-skip binary search (related)

- **Link:** [https://commitfest.postgresql.org/patch/2731/](https://commitfest.postgresql.org/patch/2731/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com)
- **Date:** decided 2020-09-15; started 2020-09-11
- **Outcome:** Withdrawn (cf)
- **Who objected:** Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Geoghegan noted that his earlier patch for the same idea, dynamic prefix truncation, hit a problem with concurrent page deletions. A scan that carries a skipped prefix down the tree can land on a page that no longer shares that prefix. The author agreed this makes the approach near impossible without resetting the prefix on every page, and withdrew the patch.
- **Reason (quote):** "Concurrent page deletions could occur, which could cause a scan that skips a prefix to miss that the page it landed on doesn't have the same common prefix anymore." ([message](https://www.postgresql.org/message-id/CAH2-WznPxocJKaz%3DXMqKqPauawi-vtB62oXT9SkHhDNb7cpV9Q%40mail.gmail.com))
- **What is different this time:** P4 carried a known-equal prefix down the tree, which concurrent page deletion breaks. This proposal codes items only against anchors on the same leaf, so that failure does not apply.


## 3. Objections a reviewer is likely to raise

### 1. On-disk format would be fixed while its basic layout is still in flux

fixable · level 2, rubric item 6 · on-disk format / pg_upgrade, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

Release 20 ships the v10 layout: 6-byte code header, MAXALIGNed items, byte-order sharing. Release 21 adopts unaligned packing or normalized keys for bigint and timestamptz. Every later release must then read both layouts in pg_upgraded indexes, with only page flags to tell them apart.

Message #9 says the compressed representation is supported forever. The format has already been replaced once, from v8 to v9. Message #15 still lists unaligned packing and a smaller header as open, and message #13 leaves normalized keys open. Message #14 offers a different zone-based layout, and message #15 says neither layout is plainly right.

Thread: [#9 (Peter Geoghegan, 2026-11-03)](https://www.postgresql.org/message-id/CAH2-WznyrHYLaROHDtaZfYWn3u6hx7jpzsbpQO_vxk8jDFO8GA%40mail.gmail.com), [#13 (Greg Burd, 2026-11-09)](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki), [#14 (Yao Feng, 2026-11-10)](https://www.postgresql.org/message-id/CAAU0Mh1an_cU-MoTgwfagxn1Bf0v8qc%2BPD-2WnkUvVJKZOFPng%40mail.gmail.com), [#15 (Greg Burd, 2026-11-10)](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki)

**Question:** Which of the open format questions (header size, unaligned packing, normalized keys) will be settled before commit, and how would a later change be marked on disk?

Counter-check: held. Messages #13 and #15 leave header size, unaligned packing and normalized keys open, and message #15 says neither layout is plainly right. The proposal does not say how a later layout change would be marked on disk.

### 2. No stated guarantee that a coded leaf can always be split

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-3`

A user inserts a large key into a full coded leaf whose decoded contents are near the two-page limit. The retry path, which splits the existing items alone, still finds no legal split point. The insert then fails with an error on that page and keeps failing until the index is rebuilt, and the same layout may also be unreplayable on a standby.

Peter asked in message #9 for guarantees that a legal split is always available in pathological cases, and called it a property of the on-disk representation. The v10 commit message describes a cached decoded-size check plus an "existing items are split alone and the insert is retried" path, but it gives no argument for why the retry always succeeds. Message #3 also notes that the forced-image fallback is reachable only through an injection point. Peter has not replied since v9.

Thread: [#3 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/y8lIX15Mjse_ALeMMfsK1oakyoxjIU4p2HLHF8sQArBsYgFDvAZz7vVCW26vl6IP7AEMWQArSkqBxvyCa4OKdXe8eiZ6OxqgvIC0FnJEM7c%3D%40burd.me), [#9 (Peter Geoghegan, 2026-11-03)](https://www.postgresql.org/message-id/CAH2-WznyrHYLaROHDtaZfYWn3u6hx7jpzsbpQO_vxk8jDFO8GA%40mail.gmail.com), [#13 (Greg Burd, 2026-11-09)](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki)

**Question:** What bound on decoded page contents, enforced on every insert, dedup and redo path, proves that both a split and its retry always find a legal point?

Counter-check: held. The README points to an argument in _bt_prefix_split_needed for the balanced split, but the visible material gives none for why the retry path always ends. The forced-image fallback in message #3 belongs to the replaced design and must not be used as support.

### 3. Redo reruns the grouping heuristic, freezing it into the WAL format

fixable · level 2, rubric item 9 · WAL / replication / standby, backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A minor release changes how leaf groups are chosen, as v10 already did. The standby runs the new minor version and the primary the old one, the usual order during a minor upgrade. Redo lays out a different coded left half than the primary wrote, and with wal_consistency_checking off nobody notices until a later read or failover returns wrong results.

The v10 commit message says redo collects the left half's items as the primary did and lays them out with the same routine, so the grouping rule runs on both sides. Message #2 names this hazard: two copies that must agree exactly, or a standby diverges with no version bump to catch it. Message #15 changed that rule between v9 and v10. A shared routine removes the duplicated code, but every future fix to the heuristic becomes a WAL format change that is easy to miss in a backpatch.

Thread: [#2 (Greg Burd, 2026-10-27)](https://www.postgresql.org/message-id/iMzeiKz_iFUwu8k8FTdo2CJ1Qp_x7SkvdLxEtYPhM3y_XfFY8mNM1fpSKoWimjR9ZKTz_4XT8BNCYKJWSq9w087DmY29cvnVrmvKCzLplnk%3D%40burd.me), [#15 (Greg Burd, 2026-11-10)](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki)

**Question:** Why not log the chosen groups for both halves, so a later change to the grouping rule cannot make primary and standby build different pages?

Counter-check: held. The v10 commit message says redo lays out the left half with the same routine and does not say the groups are logged. Existing nbtree redo logs decisions such as dedup intervals and replays only mechanical steps, so this departure needs a stated reason.

### 4. Every future leaf reader must use _bt_getitem, and a missed one stays silent

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A later nbtree patch, or an extension that inspects btree pages, reads a leaf data item with PageGetItem as all existing code does. On plain pages it works and its tests pass. On a BTP_PREFIX page it reads a coded tuple as a real key, which shows up in the field as a wrong comparison or a wrong index-only scan result.

Message #1 says every read of a leaf data item must go through _bt_getitem, and the audit missed the array-scan look-ahead in _bt_readpage on its first pass. Old and new formats coexist in one index with no version bump, so this rule applies to all nbtree code from now on. The proposal has no check that fails loudly when PageGetItem returns a coded item. External tools that parse btree pages get no signal beyond a flag bit.

Thread: [#1 (Greg Burd, 2026-10-26)](https://www.postgresql.org/message-id/at-01F1wsXLRH0qo%40floki)

**Question:** What makes a leaf data item read that bypasses _bt_getitem fail loudly in assert builds, and what is the plan for out-of-tree code that reads btree pages?

Counter-check: held. Posting list tuples carry a similar rule, but they are marked in the tuple and exercised by default. A coded item has no tuple-level marker and the parameter is off by default, so a missed reader passes the regression tests.

### 5. No master-versus-patch numbers for the default-off path in v10

fixable · level 2, rubric item 8 · performance on the hot path · `prior-attempts-1`

A user who never enables prefix_compression upgrades. Every leaf read now checks BTP_PREFIX and goes through _bt_getitem(), and _bt_compare() decodes on compressed pages. If that adds a few percent to descent or scans, every btree index pays for it, and users see it only as slower queries after the upgrade.

Message #1 showed the off path matching master, but that was for v1, which v9 and v10 replaced (message #13). The v10 commit message compares the index only with itself without the parameter, in the same binary. [P2] was returned because of about 2% insert and 15% reindex regressions on common multi-column indexes. Reader paths that every index uses carry the same risk here.

**Precedents.**
- [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (Returned with feedback, 2022-02-07); thread: [link](https://www.postgresql.org/message-id/flat/CAEze2Whwvr8aYcBf0BeBuPy8mJGtwxGvQYA9OGR5eLFh6Q_ZvA@mail.gmail.com)

Thread: [#1 (Greg Burd, 2026-10-26)](https://www.postgresql.org/message-id/at-01F1wsXLRH0qo%40floki), [#13 (Greg Burd, 2026-11-09)](https://www.postgresql.org/message-id/avHtA-ipf4e30Hfk%40floki), [#15 (Greg Burd, 2026-11-10)](https://www.postgresql.org/message-id/avNYO5QY5Jl127Rm%40floki)

**Question:** With prefix_compression off, how do v10 point lookups, range scans, inserts and CREATE INDEX compare with master on common single- and multi-column indexes?

Counter-check: held. The only off-versus-master numbers are for v1 and v4, and message #13 says v9 replaces that design completely. [P2] concerns a different patch with a measured regression, so it is an analogy only and must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New BTP_PREFIX leaf format with a prefix directory item coexists with plain pages in one index indefinitely. |
| WAL / replication / standby | **yes** | xl_btree_split gains a flags field, XLOG_PAGE_MAGIC is bumped, and redo rebuilds coded pages without full-page images. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Out-of-tree code that reads leaf data items with PageGetItem gets coded bytes on BTP_PREFIX pages. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The new prefix_compression storage parameter must be dumped and restored like other btree reloptions. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Redo reruns the layout routine, so a later fix to grouping changes what a standby on another minor version builds. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second permanent leaf format, and every future leaf reader must go through _bt_getitem(). |
| performance on the hot path | **yes** | Every leaf read checks the page flag and goes through _bt_getitem(), also for indexes with the parameter off. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which open format questions (header size, unaligned packing, normalized keys for fixed-width types) will be settled before commit, and how would a later layout change be marked on disk?
2. What bound on decoded page contents, kept by every insert, deduplication and redo path, proves that a split and its retry always find a legal split point?
3. Why does split redo rerun the layout routine for the left half instead of replaying logged groups and anchors for both halves?
4. With prefix_compression off, how does v10 compare with master on point lookups, range scans, inserts and CREATE INDEX for common single-column and multicolumn indexes?
5. Which real schemas, with their index definitions, have btree keys whose stored bytes share long prefixes, and how much do those indexes shrink with v10?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Geoghegan asked the design questions in message #9 but has not replied since v9. The owner must know nbtree page splits, deduplication and nbtree redo.
- **Liability:** A second leaf format that every later release must read, because coded pages are never converted back. Also the coded-page redo paths and the rule that all leaf readers use _bt_getitem().
- **Author follow-up:** The author posted ten versions in about three weeks and says in message #1 that he will work through whatever review turns up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Keys with long shared prefixes and low-cardinality leading columns are common enough to matter. The patch is about 175 KiB in one commit across nbtree insert, split, deduplication, search and redo, and the permanent format is still open.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-4` (fixable, level 2) Large single-commit nbtree change without an engaged owner for the aftermath. Not listed: not among the strongest objections.
- `demand-1` (fixable, level 1b) Motivating jsonb workload is asserted, and expression index keys may not hold paths. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` Contradictory statements on BTREE_VERSION and catalog version bumps. Refuted (already handled): Message #13 says v9 replaces v1-v8 completely and that BTREE_VERSION and the catalog do not change. The v10 commit message repeats this and names XLOG_PAGE_MAGIC as the only bump, so the current design is consistent.
- `standards-compat-3` pageinspect output changes meaning on compressed leaves. Refuted (already handled): The pageinspect doc hunk states where the directory item sits, that BTP_PREFIX marks the page, and that itemlen is the stored size. How bt_page_stats counts the directory is line-level detail, outside levels 1-2.
- `replication-ondisk-1` Split and dedup redo recompute layout, so minor-release fixes can desynchronize standbys. Refuted (not specific): Duplicate of maintenance-fit-1. Logging the chosen groups for both halves would solve it, so the severity is fixable in any case.
- `replication-ondisk-2` Leaf item reads through PageGetItem silently return coded bytes. Refuted (not specific): Duplicate of maintenance-fit-2.
- `maintenance-fit-3` Permanent on-disk format proposed before the design space is settled. Refuted (not specific): Duplicate of standards-compat-1, which covers the same open format questions and also asks how a later change would be marked on disk.
- `already-exists-1` No comparison with hash and GIN indexes for the motivating workloads. Refuted (no failure scenario): Hash indexes are single-column and cannot serve unique constraints, ordering or prefix range scans. GIN on jsonb serves containment only. The tenant-prefixed multicolumn keys, file paths and unique URL keys named in the thread need btree, as the objection itself concedes.
- `demand-2` Measured gain on realistic keys is modest beside a 10% write cost. Refuted (already handled): The 8-10% table is from the replaced v4 design on near-duplicate keys, which message #4 explains; v10 reports 35-52%. The 10% write cost falls only on indexes that turn the parameter on, so rubric item 4 does not fail.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent tried storage-level prefix compression in nbtree. Peter's own unposted prototype from April (message #9) is the closest earlier attempt, and the thread does not say why it stalled.
- standards-compat: 3 objection(s). The reloption itself follows the existing deduplicate_items pattern, and its pg_dump round-trip behaves as other btree reloptions do, so no objection is raised on that point.
- replication-ondisk: 3 objection(s). The nbtree VACUUM bookkeeping invariant does not apply: emptying a coded leaf drops its directory and leaves no state for a later VACUUM pass.
- security: 0 objection(s). The proposal has no privilege aspect. prefix_compression is an index storage parameter that only the index owner can set, it is off by default, and it changes no SQL-visible behaviour: comparisons, RLS, column privileges and leakproofness all see the same decoded values. A size side channel, where a user inserts guessed keys and watches index growth, already exists through default-on deduplication and is coarser at page granularity. Memory checks, decode bounds and redo safety on corrupt pages are not security questions at design level and need the code to check.
- maintenance-fit: 4 objection(s). Checked the nbtree VACUUM bookkeeping invariant; the proposal leaves no state for a later VACUUM pass, since an emptied coded leaf drops its directory, so it does not apply.
- already-exists: 1 objection(s). No extension or contrib module offers btree leaf prefix compression, and nbtree's leaf format cannot be changed from outside core. The only out-of-core alternative named is the author's own separate index AM, BARK.
- demand: 2 objection(s). Oracle and SQL Server have prefix compression, and message #9 accepts that the technique works in sympathetic cases. The objections above are about how large the group that gains is.

### Precedents retrieved

- P1: [nbtree: implement dynamic prefix truncation](https://commitfest.postgresql.org/patch/4635/) (cf, Withdrawn, 2026-10-01) - used in the review; reason summarised from its thread
- P2: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P3: [Btree compression](https://commitfest.postgresql.org/patch/494/) (cf, Returned with feedback, 2016-03-29) - used in the review; reason summarised from its thread
- P4: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - used in the review; reason summarised from its thread
- P5: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant; reason summarised from its thread
- P6: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P7: [Revert "Fix bogus %name-prefix option syntax in all our Bison files."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71ed8b3ca76cd720f4013c3c20f0d2706583ab9a) (revert, Reverted, 2014-05-28) - judged not relevant
- P8: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P9: [Revert idea of zer-padding padding session id in log_line_prefix](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b06e6aba8773df8040b357a1cb058879ed2d94f) (revert, Reverted, 2013-05-06) - judged not relevant
- P10: [Revert thinko introduced into prefix_selectivity() by my recent patch: make_greater_string needs the < procedure not the >= one. Spotted by Peter.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=164899db1ceed1e582e5ae1af9455740e7aaa94a) (revert, Reverted, 2008-03-17) - judged not relevant
- P11: [Leaf and internal block access stats for indexes](https://commitfest.postgresql.org/patch/5614/) (cf, Withdrawn, 2026-04-10) - judged not relevant; reason summarised from its thread
- P12: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P13: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P14: [nbtree performance improvements through specialization on key shape](https://commitfest.postgresql.org/patch/3672/) (cf, Withdrawn, 2026-10-01) - judged not relevant
- P15: [Adding a { LOCK ROWS | LOCK KEY INDEX } clause to FK definitions](https://commitfest.postgresql.org/patch/6923/) (cf, Withdrawn, 2026-09-09) - judged not relevant
- P16: [Fix pg_stat_statements display of normalized FETCH counts](https://commitfest.postgresql.org/patch/6756/) (cf, Withdrawn, 2026-05-12) - judged not relevant
- P17: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant
- P18: [Revert "Improve compression and storage support with inheritance"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=74563f6b90216180fc13649725179fc119dddeb5) (revert, Reverted, 2024-02-20) - judged not relevant
- P19: [Page compression for OLTP](https://commitfest.postgresql.org/patch/3783/) (cf, Returned with feedback, 2023-01-31) - used in the review
- P20: [WAL compression setting after PostgreSQL LZ4 default change](https://commitfest.postgresql.org/patch/6979/) (cf, Withdrawn, 2026-07-03) - judged not relevant
- P21: [sort leaf pages by ctid for gist indexes built using sorted method](https://commitfest.postgresql.org/patch/3454/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P22: [PGSERVICEFILE as part of a normal connection string \[duplicated please delete\]](https://commitfest.postgresql.org/patch/5659/) (cf, Withdrawn, 2025-03-20) - judged not relevant
- P23: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P24: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 219 s; cost reported by the CLI: $4.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 124 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:33.883Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
