# Design pre-review: GIN pageinspect support for entry tree and non-leaf posting tree

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6128](https://commitfest.postgresql.org/patch/6128/) (Needs review); authors: Kirill Reshke (reshke)
- Thread: [GIN pageinspect support for entry tree and posting tree](https://www.postgresql.org/message-id/flat/CALdSSPiN13n7feQcY0WCmq8jzxjwqhNrt1E%3Dg%3Dg6aZANyE_OoQ%40mail.gmail.com), started 2025-10-13
- Messages read: 46 of 46
- Patch prose read: v20260308-0003-Move-IS_INDEX-macro-to-pageinspect.h.patch, v20260308-0001-Modernize-coding-in-GIN-pageinspect-functi.patch, v20260308-0002-GIN-pageinspect-support-for-entry-tree-and.patch, v1-0001-GIN-pageinspect-support-for-entry-tree-and-postin.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, the cost stays inside pageinspect, and the page-level shape answers the concern recorded in P1. Two fixable level 2 points remain, standards-compat-1 and standards-compat-2, both about output columns that pageinspect 1.14 would fix as a user-facing contract. A committer would likely want them settled before commit, but neither requires a different proposal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so safety on corrupted pages, the privilege setup in the 1.13--1.14 script and the actual NULL category output could not be checked. Only 3 of 46 messages were visible, so some open points may have been answered in the thread.

## The proposal, restated

**Problem.** pageinspect can decode only GIN posting tree leaf pages, through gin_leafpage_items. Users who study GIN storage or fix corrupted GIN indexes cannot see entry tree pages or posting tree internal pages. The author cites education and corruption repair as uses.

**Proposed design.** The patch adds two functions in a new pageinspect 1.14 extension version. gin_entrypage_items decodes entry tree leaf and internal pages, including keys of multi-column indexes, and gin_datapage_items decodes posting tree internal pages. It follows the one-function-per-page-type style of the hash and BRIN functions and rejects metapages, fast list pages and data pages with an error.

**User-visible surface.**
- New function gin_entrypage_items(page bytea, reloid oid) returns setof record with columns itemoffset, downlink, tids, keys.
- New function gin_datapage_items(page bytea) returns setof record with columns itemoffset, downlink, item_tid.
- pageinspect extension version moves from 1.13 to 1.14.
- New error messages when either function gets a metapage, fast list page, data page or non-GIN index.

**What the author claims.**
- The functions help with education and with corruption repair on GIN indexes.
- Separate functions per page type match the existing hash and BRIN pageinspect design.
- The patch handles NULL keys after a fix found in review, with tests added.
- Output differs between 32-bit and 64-bit builds because GIN aligns the posting list offset on the platform, so an alternative expected file covers 32-bit.

**Thread state.** Andrey Borodin, Peter Eisentraut, Chao Li, Japin Li and Roman Khapov reviewed it, and the entry reached Ready for Committer, then went back for a missing TupleDescFinalize() call after commit c456e391138. Japin Li posted a rebased v20260901 with that fix, and no committer has replied yet.

## 2. Prior attempts

### Pageinspect - add functions on GIN and GiST indexes from gevel (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1207/](https://commitfest.postgresql.org/patch/1207/)
- **Thread:** [https://www.postgresql.org/message-id/flat/accae316-5e4d-8963-0c3d-277ef13c396c@postgrespro.ru](https://www.postgresql.org/message-id/flat/accae316-5e4d-8963-0c3d-277ef13c396c@postgrespro.ru)
- **Date:** decided 2017-11-27; started 2017-07-21
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Alexander Korotkov, Tomas Vondra, Ashutosh Sharma, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers said the functions read the whole index, while pageinspect inspects single pages, so the code belongs in a new extension or should be rewritten as page-level functions. Peter Eisentraut also found that the patch no longer compiled and that the stats functions returned one formatted text value where they should return a row with columns. The author planned a new contrib module but posted no new version, and the entry was returned with feedback after three weeks of waiting on the author.
- **Reason (quote):** "The proposed patch uses a very different approach, reading the whole index, not individual pages. Why should it be part of pageinspect?" ([message](https://www.postgresql.org/message-id/2514c060-717e-6fb9-981c-bf1344174da5%402ndquadrant.com))
- **What is different this time:** P1 moved gevel's whole-index functions into pageinspect. This proposal decodes one raw page per call through get_raw_page, which is the page-level shape the P1 reviewers asked for.

### Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row (related)

- **Link:** [https://commitfest.postgresql.org/patch/3713/](https://commitfest.postgresql.org/patch/3713/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANugjht-=oGMRmNJKMqnBC69y7vr+wHDmm0ZK6-1pJsxoBKBbA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANugjht-=oGMRmNJKMqnBC69y7vr+wHDmm0ZK6-1pJsxoBKBbA@mail.gmail.com)
- **Date:** decided 2022-10-12; started 2022-06-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane listed open issues before the patch could be committed: no documentation update, no simple way to ask for all remaining pages (a count one too large raises an error), and a regression test that depends on tuple layout. He then showed the test fails on a 32-bit machine. The author did not address this feedback, and Michael Paquier marked the entry Returned with feedback after four weeks of inactivity.
- **Reason (quote):** "This feedback has not been addressed and the thread is idle four weeks, so I have marked this CF entry as RwF." ([message](https://www.postgresql.org/message-id/Y0ZV%2BV%2FkDk4ygdPm%40paquier.xyz))
- **What is different this time:** P5 stalled partly on a layout-dependent test that failed on 32-bit. This patch ships a 32-bit expected file (gin_1.out), and the thread accepts the 8 KiB block size dependency.


## 3. Objections a reviewer is likely to raise

### 1. Output columns encode on-disk details that later fixes cannot change

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A user reads downlink (2147483696,3) on an entry leaf page as a block number and goes looking for block 2147483696. A script that compares output across machines gets a different value on 32-bit builds. Once pageinspect 1.14 ships, changing these columns needs another extension version and an upgrade script.

Message #1 says the leaf downlink has a different meaning from the internal one but is shown the same way, and message #10 confirms it varies by platform. The column named downlink is a tid in gin_entrypage_items but an integer in gin_datapage_items. gin_entrypage_items takes reloid oid, while gist_page_items takes regclass. The doc hunk shows examples only and gives no column definitions.

Thread: [#1 (Kirill Reshke, 2025-10-13)](https://www.postgresql.org/message-id/CALdSSPiN13n7feQcY0WCmq8jzxjwqhNrt1E%3Dg%3Dg6aZANyE_OoQ%40mail.gmail.com), [#10 (Kirill Reshke, 2026-01-06)](https://www.postgresql.org/message-id/CALdSSPgxaLX-%3DyXm-mVF47S7fYJAaEoqycXw0G3yLS5wH%3D019w%40mail.gmail.com), [#11 (Kirill Reshke, 2026-01-06)](https://www.postgresql.org/message-id/CALdSSPjX0rpG%2BWDf8CFE_jZGFOAFvx5uoihjPrXTpjJJvMfGeA%40mail.gmail.com)

**Question:** Can leaf tuples report a decoded posting list offset and item count, or a posting tree root block, instead of the raw tid, and why does downlink differ in type between the two functions?

Counter-check: held. Message #1 confirms the leaf downlink is a raw encoded value shown as a tid, and message #10 confirms it varies by platform. The doc hunk defines no columns, and no reviewer raised the column shape, so the thread does not answer this.

### 2. keys column format and NULL categories are not defined

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user repairing a corrupted index sees y=NULL and cannot tell a NULL array element from a NULL array or an empty array, which GIN stores as different key categories. Tools that parse keys from gist_page_items cannot reuse that code, because the format differs.

GIN stores separate null categories, and message #35 shows only the planned output y=NULL. The material does not say whether the keys column keeps the categories apart. gist_page_items prints keys in the index value description style, and this proposal adds a new attname=value text form without saying why.

Thread: [#29 (Roman Khapov, 2026-01-14)](https://www.postgresql.org/message-id/0A0E3993-7617-48EA-8555-6C1907C147E5%40yandex-team.ru), [#35 (Kirill Reshke, 2026-01-14)](https://www.postgresql.org/message-id/CALdSSPjgB2%3Dcn2KsMSYD%3D55ft1kWv0xThNT7%2BeDHmPtGOuKo5w%40mail.gmail.com)

**Question:** How does the keys column show each GIN null category and multi-column keys, and why not use the key format gist_page_items already prints?

Counter-check: held. GIN stores three null categories, and the thread and doc hunk show only y=NULL without saying whether the keys column tells them apart. The gist_page_items format comparison is weak support, because pageinspect output formats already differ by access method.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | pageinspect moves from 1.13 to 1.14 with two new SQL functions, and patch 0003 moves the IS_INDEX macro to pageinspect.h. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Output columns encode on-disk details that later fixes cannot change |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The functions decode raw pages and run type output functions on keys; the thread does not show whether the 1.13--1.14 script revokes execute from PUBLIC. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The output columns become a contract, and the tests need a 32-bit expected file and assume the 8 KiB block size. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can gin_entrypage_items report the posting list offset, item count and posting tree root block for leaf tuples as decoded values instead of a raw tid such as (2147483696,3)?
2. Why is the downlink column a tid in gin_entrypage_items and an integer in gin_datapage_items?
3. How does the keys column show each GIN null category (null key, empty item, null item), and can a user tell them apart?
4. Will the documentation define each output column, including what tids and downlink mean on leaf pages versus internal pages?
5. Has anyone run both functions on deliberately corrupted pages, given that corruption repair is a stated use?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the GIN page layout and pageinspect conventions would have to own it. Peter Eisentraut is credited as co-author in the commit message, but the brief says no committer has replied to the latest version.
- **Liability:** The owner signs for the output columns of two functions in pageinspect 1.14, for decoding of possibly corrupted pages without crashes, and for tests that depend on platform alignment and block size. Later changes to the columns need a new extension version.
- **Author follow-up:** The author posted revisions over several months and moved the entry across CommitFests. Japin Li, a reviewer, posted the latest rebase with the TupleDescFinalize() fix.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch fills a gap in pageinspect's GIN coverage for people who study or repair GIN indexes, and the cost stays in one contrib module. It reached Ready for Committer after five reviewers and is now waiting for a committer, with two output-shape points still open.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Leaf entry page downlink column exposes raw encoded values. Refuted (not specific): Duplicate of standards-compat-1, which covers the same raw leaf downlink and also the type difference between the two functions.

### Review angles

- prior-attempts: 0 objection(s). No precedent reason still applies. P1 died over whole-index reads, and this proposal is page-level. The P5 concern about 32-bit test layout was handled in messages #10 and #40. P3 has no recorded reason. The thread cites no earlier discussion of GIN entry tree inspection.
- standards-compat: 2 objection(s). No conflict with the SQL standard, pg_dump or GUCs. The proposal adds contrib functions under a normal extension version bump, and the one-function-per-page-type split follows the existing hash and BRIN functions.
- replication-ondisk: 0 objection(s). The proposal adds two read-only pageinspect functions and an extension version bump. It changes no on-disk format, WAL record, catalog, wire protocol or core ABI, so it needs nothing from pg_upgrade or replication. The 32-bit and 64-bit output difference comes from the existing GIN page layout, which the patch only reads. Whether decoding corrupted pages is safe depends on the code and was not checked.
- security: 0 objection(s). The proposal adds no new privilege path that I can find. pageinspect is superuser-only by convention, and both functions take raw pages from get_raw_page, which already checks privileges and blocks other sessions' temp tables (message #4). The thread does not show whether the 1.13--1.14 script revokes execute from PUBLIC. gin_entrypage_items also runs type output functions on keys read from a page that may be corrupted, which gist_page_items already does. A committer would likely check both points in the code, which I cannot see.
- maintenance-fit: 1 objection(s). From the maintenance and fit angle the proposal is sound. It is page-level, unlike [P1], it costs nothing for users who do not install it, and it follows the existing hash and BRIN style. Peter Eisentraut contributed code, but no committer has taken ownership.
- already-exists: 0 objection(s). No existing Postgres feature decodes GIN entry tree pages or posting tree internal pages. pageinspect's gin_metapage_info, gin_page_opaque_info and gin_leafpage_items cover other page types, and amcheck's GIN check validates pages without showing their contents. The gevel extension in [P1] works on whole indexes, not single pages, so it does not do this job, and the patch extends pageinspect's existing per-page-type mechanism without adding a second one.
- demand: 0 objection(s). No demand objection. pageinspect already decodes every page type for btree, hash, BRIN and GiST, so filling the GIN gap matches existing coverage, and Andrey Borodin called it useful in message #3. The cost stays inside one contrib module and reaches only people who install it; the corruption-repair use is asserted, not shown.

### Precedents retrieved

- P1: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - used in the review; reason summarised from its thread
- P2: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P3: [pageinspect: Hash index support](https://commitfest.postgresql.org/patch/758/) (cf, Returned with feedback, 2016-12-20) - used in the review
- P4: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P5: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P6: [Fixes a trivial bug in dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5996/) (cf, Withdrawn, 2025-09-02) - judged not relevant; reason summarised from its thread
- P7: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P8: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant; reason summarised from its thread
- P9: [new diagnostic items for the dynamic sql](https://commitfest.postgresql.org/patch/3537/) (cf, Returned with feedback, 2022-12-14) - judged not relevant; reason summarised from its thread
- P10: [brin: avoid errors processing work item following concurrent reindex](https://commitfest.postgresql.org/patch/2838/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P11: [KNN for B-tree](https://commitfest.postgresql.org/patch/1804/) (cf, Returned with feedback, 2020-03-16) - judged not relevant
- P12: [proposal of adding 2 new diag items related to parsing the dynamic sql statement](https://commitfest.postgresql.org/patch/3258/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P13: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant
- P14: [btree_gin and ranges](https://commitfest.postgresql.org/patch/24/) (cf, Rejected, 2015-02-03) - judged not relevant
- P15: [Clarify ItemPointers for Table AM API](https://commitfest.postgresql.org/patch/3108/) (cf, Withdrawn, 2021-06-18) - judged not relevant
- P16: [Sequence Access Method](https://commitfest.postgresql.org/patch/44/) (cf, Returned with feedback, 2015-07-28) - judged not relevant
- P17: [Reloptions for table access methods](https://commitfest.postgresql.org/patch/2717/) (cf, Withdrawn, 2021-04-06) - judged not relevant
- P18: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P19: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P20: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Sequence Access Methods](https://commitfest.postgresql.org/patch/466/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P22: [VACUUM (FAST_FREEZE )](https://commitfest.postgresql.org/patch/2908/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P23: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P24: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 119 s; cost reported by the CLI: $3.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 6 verified, 1 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:43:07.206Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
