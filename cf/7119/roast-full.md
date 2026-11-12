# Design pre-review: Support specialized B-tree page searches

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7119](https://commitfest.postgresql.org/patch/7119/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Support specialized B-tree page searches](https://www.postgresql.org/message-id/flat/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), started 2026-08-05
- Messages read: 3 of 3
- Patch prose read: v1-0002-Use-interpolation-for-specialized-int4-page-searc.patch, v1-0001-Add-B-tree-support-for-specialized-page-searches.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is stated and the benchmarks are detailed, but the design of support procedure 7 is not settled. The int4 callback copies nbtree search rules (maintenance-fit-1), and the two documentation hunks describe different contracts (standards-compat-1). The thread does not measure the existing SortSupport comparator as an alternative (already-exists-1). Message #3 proposes a different split and has no answer, so the list has to agree on the callback contract before code review makes sense.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check that the interpolation probe stays inside the page's line pointer range on a corrupted page with unsorted keys. The cost of the new path for opclasses without support procedure 7 is also unchecked.

## The proposal, restated

**Problem.** B-tree page searches call the comparison function through fmgr and deform each index tuple. For simple fixed-width types such as int4, this overhead costs much more than the comparison itself, and it slows both searches and random insertions.

**Proposed design.** A new optional B-tree support procedure 7 lets an opclass install C callbacks that compare an insertion key with a page tuple and search a whole page interval. The callback may decline a search, and nbtree then uses the generic code. The first patch implements this for single-column, same-type int4 indexes, and the second adds an interpolation probe that falls back to binary search unless the adjacent tuple proves the exact boundary.

**User-visible surface.**
- New optional B-tree support function number 7 (binsearchsupport), returning the address of a C-callable page search function.
- New built-in function btint4binsearchsupport(internal), registered in the int4_ops opclass.
- C API in nbtree.h for opclass callbacks that compare keys and search a page interval.
- CREATE/ALTER OPERATOR FAMILY accept support function 7 for btree; opclasscmds.c and nbtvalidate.c change.

**What the author claims.**
- One million int4 index-only lookups on dense keys ran 35% faster (2.245 s to 1.470 s); the author calls this a favorable case.
- Specialization alone gains about 3% to 12% across all tested distributions; interpolation adds up to 34% for near-linear keys.
- Interpolation costs about 2% for probes outside the indexed range; regular pgbench point lookups gain about 2.5% to 3%.
- Duplicate lookups with 16 clients regressed by about 1.3%; the author is investigating.
- Text and UUID showed no convincing gain, so the author leaves them out.

**Thread state.** Only the author and Yao Feng replied. Yao Feng posted patches that move the search loop into nbtsearch.c on the SortSupport model and add int8 support; the author has not answered, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

### Delaying/avoiding BTreeTupleGetNAtts() call within _bt_compare() (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2429/](https://commitfest.postgresql.org/patch/2429/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAH2-Wzngz5MrkiTaZNny0GzfTxNQE+QWPPaO-C390BgruhgjEw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAH2-Wzngz5MrkiTaZNny0GzfTxNQE+QWPPaO-C390BgruhgjEw@mail.gmail.com), [https://www.postgresql.org/message-id/flat/2ca215d085a74e2eabd31b76e97cc9f3@opammb0561.comp.optiver.com](https://www.postgresql.org/message-id/flat/2ca215d085a74e2eabd31b76e97cc9f3@opammb0561.comp.optiver.com), [https://www.postgresql.org/message-id/flat/eca8c912f14c41e3955066281dcc6c5d@opammb0561.comp.optiver.com](https://www.postgresql.org/message-id/flat/eca8c912f14c41e3955066281dcc6c5d@opammb0561.comp.optiver.com)
- **Date:** decided 2020-11-09; started 2020-01-26
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Andres Freund suggested loading ntupatts where the code loads it today and changing only the loop termination, because the delayed load and extra branches could add stalls. Peter Geoghegan agreed to rework the patch that way, and the thread has no later message before the November 2020 withdrawal.
- **Reason (quote):** quote unavailable
- **What is different this time:** P4 cut per-call overhead inside the generic _bt_compare(). This proposal skips _bt_compare() for int4 through an opclass callback that reads the key directly and may add an interpolation probe.

### _bt_binsrch* improvements - equal-prefix-skip binary search (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2731/](https://commitfest.postgresql.org/patch/2731/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAEze2WjnrkzT9tfqFhSCPeH6LV-UUj5_oO5vX038dvfK7fDuyQ@mail.gmail.com)
- **Date:** decided 2020-09-15; started 2020-09-11
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P15 changed the generic _bt_binsrch loop to skip equal prefixes. This proposal keeps the generic loop and adds an opt-in per-opclass search for single-column, same-type keys.


## 3. Objections a reviewer is likely to raise

### 1. Search control flow copied into per-opclass callbacks in nbtcompare.c

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch changes how nbtree handles posting lists, pivot tuples or heap TID tie-breaking in nbtsearch.c. The int4 callback, and the int8 one proposed in message #3, keeps the old rules, and no test fails. Users of specialized opclasses get wrong search bounds only on pages that hit the changed case.

The commit message of 0001 says the int4 callback itself preserves NULLs, descending order, pivot tuples, heap TID tie breakers and posting lists. Message #3 confirms that nbtcompare.c duplicates the binary search loop, slot probing, TID tie-breaking and posting list traversal. Each added type then copies nbtree search rules again. Message #3 proposes keeping the loop in nbtsearch.c, but the author has not replied and no new version exists.

Thread: [#1 (Andrey Borodin, 2026-08-05)](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#3 (Yao Feng, 2026-10-05)](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com)

**Question:** Will the next version keep all search control flow in nbtsearch.c, so that a type callback holds only value extraction, comparison and interpolation math?

Counter-check: held. The 0001 commit message says the int4 callback itself handles pivot tuples, heap TID tie breakers and posting lists. Message #3 confirms the duplicated loop, and no version that centralizes it in nbtsearch.c is visible.

### 2. Support procedure 7 contract is not settled before it becomes permanent

fixable · level 2, rubric item 12 · ABI / extension API, catalog changes, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An extension author follows xindex.sgml and writes a support function 7 that returns a whole-page search function. A later release adopts the primitive-callback split from message #3, and the extension opclass then fails to compile or gives wrong search results. The author finds out when users upgrade.

Support procedure numbers and their signatures become a documented, catalog-visible contract in pg_amproc and in CREATE OPERATOR CLASS. The docs disagree on what that contract is: xindex.sgml says the function returns the address of one search function, while btree.sgml says it installs callbacks. The name binsearchsupport names one algorithm, but patch 0002 adds interpolation. Message #3 proposes a different callback split on the model of sortsupport and skipsupport, and the author has not answered it.

Thread: [#1 (Andrey Borodin, 2026-08-05)](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#3 (Yao Feng, 2026-10-05)](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com)

**Question:** Which callback struct will support procedure 7 fill, does it follow the sortsupport and skipsupport pattern, and is it meant for third-party opclasses?

Counter-check: held. xindex.sgml says procedure 7 returns the address of one search function, while btree.sgml says it installs callbacks. Message #3 proposes a different callback split, and the thread has no answer to it, so the contract is not settled.

### 3. Procedure 7 adds a second fast-comparator mechanism next to SortSupport

fixable · level 2, rubric item 12 · design, ABI / extension API, catalog changes, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An opclass author who wants faster index searches for int8, date or timestamp must write new procedure 7 callbacks, even though the opclass already ships an fmgr-free comparator through support procedure 2. Each new type adds about 70 lines of search callbacks. A fix to ordering rules in one mechanism can be missed in the other.

btint4sortsupport and btint8sortsupport already give fmgr-free comparators that handle reverse order and NULL placement, so the specialization part of the win could come from calling the existing SortSupport comparator in _bt_compare, plus fast extraction of a fixed-width first key. Only the interpolation in the second patch needs type-specific knowledge. Message #3 shows the current split duplicates search control flow and needs new per-type code for int8, and it asks for the SortSupport model. The author has not answered message #3.

Thread: [#1 (Andrey Borodin, 2026-08-05)](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2 (Andrey Borodin, 2026-08-07)](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru), [#3 (Yao Feng, 2026-10-05)](https://www.postgresql.org/message-id/CAAU0Mh3YyyRjSopS26N7uX4BS8tEdRMmEJ063w4%3DNNYiGdVM4g%40mail.gmail.com)

**Question:** What share of the 3-12% specialization gain remains if nbtree calls the existing SortSupport comparator, and what does procedure 7 add beyond the interpolation hook?

Counter-check: held. int4 and int8 already have fmgr-free comparators under support procedure 2, and the thread does not measure that route. The share of the 3% to 12% gain that needs a new procedure number is an open design question.

### 4. Support procedure 7 exposes nbtree page internals as an extension API

fixable · level 2, rubric item 6 · ABI / extension API, on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An extension author implements support function 7 for a custom fixed-width type, following btree.sgml. A later release changes the index tuple or posting list layout. The extension's page search compiles and runs but reads keys wrongly, and users see wrong query results after upgrading.

The documentation offers support function 7 to any operator family, and its callbacks read keys directly and search a whole page. That makes the nbtree tuple layout part of a contract with code outside core. The material does not say whether third-party opclasses are expected to use it, or how stable the callback struct is. In [P6], Tom Lane rejected a change because it gave up page layout flexibility without a compelling gain, and the same cost applies here.

**Precedents.**
- [Preventing indirection for IndexPageGetOpaque for known-size page special areas](https://commitfest.postgresql.org/patch/3543/) (Rejected, 2022-11-30); thread: [link](https://www.postgresql.org/message-id/flat/CAEze2WjE3+tGO9Fs9+iZMU+z6mMZKo54W1Zt98WKqbEUHbHOBg@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-08-05)](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2 (Andrey Borodin, 2026-08-07)](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru)

**Question:** Is support function 7 meant for third-party opclasses, and if so, which nbtree layout details does the callback API promise to keep stable?

Counter-check: held. btree.sgml offers procedure 7 to any operator family, and the material does not say which layout details the callbacks may rely on. [P6] concerned a fixed special-area offset inside core, so it should not be used as support here.

### 5. End-to-end gain is small outside server-side microbenchmarks

fixable · level 1b, rubric item 2 · demand, maintenance burden ("who maintains this in 10 years"), performance on the hot path · `demand-1`

The project adds support procedure 7, a new C callback API and a second page-search path in nbtree. Users who run ordinary OLTP queries see about 3% or nothing, and the 16-client duplicate case regresses. Future nbtree changes must keep both search paths correct for years.

The 35% to 47% figures come from loops of one million server-side lookups. Message #1 reports that standard pgbench -S showed no measurable difference, and message #2 reports about 2.5% to 3% for regular point lookups. The thread has no field report or production profile where in-page search dominates. The benefit reaches all int4 users, but its size in real workloads is not yet shown against the cost of a second search path.

Thread: [#1 (Andrey Borodin, 2026-08-05)](https://www.postgresql.org/message-id/79FF16D2-B815-45C9-A24A-D0851981E78B%40yandex-team.ru), [#2 (Andrey Borodin, 2026-08-07)](https://www.postgresql.org/message-id/2F19CB19-5FDE-4B06-9CEF-6D8075B52B21%40yandex-team.ru)

**Question:** Which real workloads, such as nested loop joins, large IN lists or random inserts, gain end to end, with profiles showing page search as a real cost?

Counter-check: held. Message #2 does report a 6% to 10% insertion gain, and the server-side loop resembles a nested loop inner scan. Still, pgbench shows about 3%, one case regresses by 1.3% without explanation, and no profile from a real workload is shown.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection maintenance-fit-2: Support procedure 7 exposes nbtree page internals as an extension API |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a pg_amproc row for support procedure 7 and a pg_proc entry for btint4binsearchsupport. |
| ABI / extension API | **yes** | nbtree.h gains a C callback API that opclasses use to compare keys and search a page interval. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | nbtree gets a second page-search path, and the int4 callback repeats pivot, heap TID and posting list rules. |
| performance on the hot path | **yes** | The patch changes the page search in nbtsearch.c; duplicate lookups with 16 clients regressed by about 1.3%. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version keep all search control flow in nbtsearch.c, as message #3 proposes, so that a type callback holds only value extraction, comparison and interpolation math?
2. Which callback struct does support procedure 7 fill, and does it return one search function (xindex.sgml) or install several callbacks (btree.sgml)?
3. What share of the 3% to 12% specialization gain remains if nbtree calls the existing SortSupport comparator and extracts the fixed-width key directly?
4. Is support procedure 7 meant for third-party opclasses, and if so, which nbtree tuple layout details does the callback API promise to keep stable?
5. What causes the 1.3% regression for duplicate lookups with 16 clients, and which real workload shows a clear end-to-end gain with a profile?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows nbtree page search, posting lists and heap TID tie-breaking; none has spoken in the visible thread.
- **Liability:** The owner signs for the correctness of a second nbtree search path and for a documented support procedure number and C callback API that stay for every later release.
- **Author follow-up:** The author posted follow-up benchmarks in message #2 and promised a rework, but has not answered message #3, and the patch needs a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** most users
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** int4 B-tree indexes are common, and the specialization gain of 3% to 12% holds across the tested distributions. The callback contract and the split between nbtsearch.c and nbtcompare.c are still open, and a review of nbtree search correctness is costly.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a rejection reason that applies here: P4 and P15 were withdrawn with no reason recorded. P6 concerns fixed special-area offsets, which this proposal does not change. Message #1 says the idea was first presented at PGConf.dev 2024, but the thread reports no discussion or objections from that talk.
- standards-compat: 1 objection(s). There is no SQL syntax, GUC or default behaviour change, and built-in opclasses are not dumped, so pg_dump round-trip is not affected for most users. Unsupported searches fall back to procedure 1.
- replication-ondisk: 0 objection(s). No objection from the replication and on-disk angle: the proposal changes no on-disk format, WAL record, wire protocol message or replicated state. It adds one pg_amproc row and one pg_proc entry, which initdb creates, and pg_upgrade needs no work because support procedure 7 is optional for user opclasses. Standbys read the same pages with the same results, and no VACUUM-time state is created, so the nbtree VACUUM bookkeeping rule does not apply. The new nbtree.h callback API is a major-release-only extension surface, not a backpatch or ABI break. Without the code, nobody can check whether the interpolation probe stays inside the page's line pointer range when a corrupted page has unsorted keys.
- security: 0 objection(s). The proposal does not change who can do what. Support procedure 7 takes and returns internal, like support procedures 2 and 6, so only a superuser can create one. Adding it to an operator class already needs superuser. Each callback reads keys from pages that nbtree already reads. Robustness against corrupt pages, such as an interpolation probe computed from damaged first and last keys, is an amcheck and implementation question for another angle. It needs the code to check.
- maintenance-fit: 2 objection(s). The nbtree VACUUM bookkeeping invariant does not apply, because the patch does not change VACUUM. Hot-path cost for non-users looks small from the prose, but the 1.3% regression with 16 clients is not explained. No nbtree committer has engaged in the visible thread, which is a committability concern only.
- already-exists: 1 objection(s). No contrib module or extension does in-page B-tree search specialization, because it needs nbtree internals. The only overlap from this angle is with SortSupport, raised above.
- demand: 1 objection(s). Demand for faster B-tree lookups on integer keys is broad, so this is a request to show real-workload gain, not a rejection.

### Precedents retrieved

- P1: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant; reason summarised from its thread
- P3: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P4: [Delaying/avoiding BTreeTupleGetNAtts() call within _bt_compare()](https://commitfest.postgresql.org/patch/2429/) (cf, Withdrawn, 2020-11-09) - used in the review; reason summarised from its thread
- P5: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P6: [Preventing indirection for IndexPageGetOpaque for known-size page special areas](https://commitfest.postgresql.org/patch/3543/) (cf, Rejected, 2022-11-30) - used in the review; reason summarised from its thread
- P7: [Floating point comparison inconsistencies of the geometric types](https://commitfest.postgresql.org/patch/679/) (cf, Returned with feedback, 2017-03-16) - judged not relevant; reason summarised from its thread
- P8: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P9: [Eliminating CREATE INDEX comparator TID tie-breaker overhead](https://commitfest.postgresql.org/patch/312/) (cf, Rejected, 2015-07-30) - judged not relevant; reason summarised from its thread
- P10: [KNN-btree](https://commitfest.postgresql.org/patch/966/) (cf, Returned with feedback, 2017-03-09) - judged not relevant
- P11: [Revert "Specialize MemoryContextMemAllocated()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fd6a44ad5744905e9ceacef7aaf698f608e6adb) (revert, Reverted, 2020-03-19) - judged not relevant
- P12: [btree_gin and ranges](https://commitfest.postgresql.org/patch/24/) (cf, Rejected, 2015-02-03) - judged not relevant
- P13: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P14: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P15: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - used in the review
- P16: [Reduce size of fmgr_builtins array by 1/3](https://commitfest.postgresql.org/patch/2411/) (cf, Rejected, 2020-01-18) - judged not relevant
- P17: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P18: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant
- P19: [SQL Inlining Behavior](https://commitfest.postgresql.org/patch/1965/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P20: [Allow excluding pg_temp or pg_catalog from search_path.](https://commitfest.postgresql.org/patch/4508/) (cf, Withdrawn, 2023-11-17) - judged not relevant
- P21: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P22: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P23: [PageRepairFragmentation optimization](https://commitfest.postgresql.org/patch/90/) (cf, Returned with feedback, 2015-07-26) - judged not relevant
- P24: [Revert renaming of int44in/int44out.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c40e20a83ce521b5b5403b08dde05b0f0641d77f) (revert, Reverted, 2018-02-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 122 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 75 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:20:50.048Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
