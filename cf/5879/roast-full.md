# Design pre-review: amcheck: support for GiST

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5879](https://commitfest.postgresql.org/patch/5879/) (Ready for Committer); authors: Andrey Borodin (x4m)
- Thread: [amcheck: support for GiST](https://www.postgresql.org/message-id/flat/5FC1B5B6-FB35-44A2-AB62-632F14E958C5%40yandex-team.ru), started 2025-06-30
- Messages read: 17 of 17
- Patch prose read: v20251216-0001-Move-normalize-tuple-logic-from-nbtcheck-t.patch, v20251216-0003-Address-review-comments-by-Mi-osz-and-Serg.patch, v20251216-0002-Add-gist_index_check-function-to-verify-Gi.patch, v2025-06-30-0002-Add-GiST-support-to-pg_amcheck.patch, v2025-06-30-0001-Add-gist_index_check-function-to-verify-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The goal fits amcheck and no objection survived at levels 1a or 1b. One fixable level 2 objection survived, prior-attempts-1: the visible thread does not state the relation lock level or explain why the parent and child recheck is correct under concurrent page deletion, the two points recorded in P3. A committer who knows GiST can settle this from a short written answer and the code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the lock level, the recheck under concurrent page deletion, behaviour on a standby and the 0003 fixes could not be checked. Only the first 3 of 17 messages were read in full.

## The proposal, restated

**Problem.** amcheck can verify B-tree and GIN indexes but has no check for GiST, so users cannot detect GiST index corruption with it. Message #1 does not say more about who needs it; it presents the patch as the next step after GIN support in PG19.

**Proposed design.** A new amcheck function walks the GiST index depth-first and checks that each downlink tuple fits inside its parent key, and that the tree is balanced. On a mismatch it locks parent and child in page-split order and rechecks, so concurrent splits do not give false reports. With heapallindexed it also checks that heap tuples are in the index, using tuple normalization moved from the B-tree checker into shared code.

**User-visible surface.**
- New SQL function gist_index_check(index regclass, heapallindexed boolean) returns void in amcheck.
- amcheck extension version 1.6, with upgrade script amcheck--1.5--1.6.sql.
- New corruption error messages, for example deleted internal page, deleted page with tuples, internal page became leaf.
- Early patch versions add GiST and GIN checks to pg_amcheck; later versions in the thread no longer include that part.

**What the author claims.**
- The GiST function is in decent shape and was only rebased from earlier work.
- Rechecking with parent and child locks makes the check safe while concurrent inserts split pages.
- A tester reported run times of about 0.3s to 14s on 10M-row indexes across 9 core opclasses, and found no problems.
- Version 0003 cuts memory for heapallindexed on 2M rows from about 300MB to about 1MB.
- Version 0003 reports a link of InvalidBlockNumber and a root page marked as split as corruption, where the check used to extend the index or crash.

**Thread state.** Several reviewers sent fixes and the CommitFest entry is Ready for Committer, but it has moved through 7 CommitFests. The latest message (2026-09-25) adds new review fixes in 0003 and asks if a committer will pick it up, and no committer has replied.

## 2. Prior attempts

### amcheck verification for GiST (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/1800/](https://commitfest.postgresql.org/patch/1800/)
- **Thread:** [https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru)
- **Date:** decided 2019-11-28; started 2018-09-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Geoghegan questioned whether the checks are safe under an AccessShareLock, since Heikki's v6 had used a ShareLock. He doubted that the parent re-find logic copied from gistfinishsplit() applies here and asked how concurrent page deletion could affect it. Andrey Borodin did not answer these questions for about two months, so Michael Paquier returned the entry with feedback.
- **Reason (quote):** "My main concern now is the heavyweight lock strength needed by the new function." ([message](https://www.postgresql.org/message-id/CAH2-WznzhHKREeXs1RMFYes%2BAo7aam5dPSmvS2Hto6WUcjEk4A%40mail.gmail.com))
- **What is different this time:** The function is rebased, with shared tuple normalization for heapallindexed, extra corruption checks and fixes for false positives. The visible thread does not say what changed about lock strength or page deletion.

### Verify true root on replicas with amcheck (related)

- **Link:** [https://commitfest.postgresql.org/patch/2418/](https://commitfest.postgresql.org/patch/2418/)
- **Thread:** [https://www.postgresql.org/message-id/flat/AAF5CE5C-5152-44D8-B437-A0784697DBA5@gmail.com](https://www.postgresql.org/message-id/flat/AAF5CE5C-5152-44D8-B437-A0784697DBA5@gmail.com)
- **Date:** decided 2020-04-08; started 2020-01-09
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped responding. Peter Geoghegan asked what true root corruption meant and what symptom it caused, and he did not want to commit a patch that holds multiple buffer locks without a strong justification. The patch also stopped applying, and David Steele returned it after no reply from the author.
- **Reason (quote):** "Your patch looks like it gets it right (it won't deadlock with other sessions that access the metapage), but I hesitate to commit it without a strong justification." ([message](https://www.postgresql.org/message-id/CAH2-WzkxnA39Rzad%2BM7DJOQ3Up%3D_7dTE-6jXmx%2B4mp%2B5v%2BKWzw%40mail.gmail.com))
- **What is different this time:** P5 was a B-tree true-root check on replicas. It is related only because that check also held two buffer locks at once and was asked for a strong justification.


## 3. Objections a reviewer is likely to raise

### 1. Lock strength and page-deletion concerns from the 2019 attempt are unanswered

fixable · level 2, rubric item 7 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A DBA runs gist_index_check on a busy production table while VACUUM deletes empty GiST leaf pages. The check follows a downlink to a deleted or reused page, or re-finds the wrong parent, and reports corruption that is not there. The DBA then rebuilds a healthy index or opens a false bug report.

[P3] is the same function by the same author, returned in 2019. The recorded concern there was the heavyweight lock level (AccessShareLock against an earlier ShareLock), whether the parent re-find logic copied from gistfinishsplit() holds here, and concurrent page deletion. None of the 17 messages states the lock level or explains why the re-check is safe against page deletion. Message #4 touches concurrency only for downlink bounds, and the commit message only says that locks follow page-split order. [P5] shows that holding several buffer locks at once was also asked to carry a strong justification.

**Precedents.**
- [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (Returned with feedback, 2019-11-28); thread: [link](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru)
- [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (Returned with feedback, 2020-04-08); thread: [link](https://www.postgresql.org/message-id/flat/AAF5CE5C-5152-44D8-B437-A0784697DBA5@gmail.com)

Thread: [#1 (Andrey Borodin, 2025-06-30)](https://www.postgresql.org/message-id/5FC1B5B6-FB35-44A2-AB62-632F14E958C5%40yandex-team.ru), [#4 (Kirill Reshke, 2025-10-22)](https://www.postgresql.org/message-id/CALdSSPhs4sCd5S_euKxTufk8sOA739ydBwhoGFYQenya7YZyiA%40mail.gmail.com), [#12 (Japin Li, 2026-01-10)](https://www.postgresql.org/message-id/MEAPR01MB30311C5AE2220142641DA0C2B683A%40MEAPR01MB3031.ausprd01.prod.outlook.com)

**Question:** Which heavyweight lock does gist_index_check take, and why is the parent and child re-check correct under that lock when VACUUM deletes GiST pages concurrently?

Counter-check: held. The visible thread and patch prose do not state the relation lock level or address concurrent page deletion, the two points recorded in [P3]. [P5] concerns a B-tree metapage check and gives only weak support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | amcheck moves to version 1.6 with amcheck--1.5--1.6.sql, and tuple normalization moves from verify_nbtree.c into verify_common. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | A new SQL function reads any GiST index and its heap; the upgrade script must set EXECUTE privileges as other amcheck functions do. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | verify_gist.c takes parent and child locks in page-split order, so it must follow future changes to GiST split and page deletion. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which heavyweight lock does gist_index_check take on the index and on the table, and does it block writes?
2. Why is the parent and child recheck correct when VACUUM deletes or recycles GiST pages during the check?
3. Does the parent re-find logic questioned in P3 still exist in the current patch, and if so, why does it hold here?
4. Is the pg_amcheck part dropped, moved to another thread, or still planned for this series?
5. Has anyone reviewed the 0003 fixes, including the downlink block-number checks from message #3 and the TAP test that corrupts the index file?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows GiST page split and page deletion locking, and the amcheck conventions for lock levels and false positives. The latest message asks for a committer and none has replied.
- **Liability:** The owner signs for the absence of false corruption reports and of deadlocks under concurrent splits and VACUUM, and for keeping verify_gist.c in step with GiST changes, with no end date.
- **Author follow-up:** The author has rebased and moved the entry through 7 CommitFests and posted new fixes on 2026-09-25, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The check fills a gap in amcheck for a widely used index type and costs nothing for users who do not call it. The entry is Ready for Committer after several reviews, but no committer has replied, and the lock-level question from P3 needs a written answer first.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Concurrency recheck relies on GiST split and deletion internals, never confirmed. Refuted (not specific): Duplicate of prior-attempts-1, which asks the same lock-level and page-deletion questions. The remaining point, that a verifier can drift from the access method code, applies equally to the existing B-tree and GIN checkers in amcheck.
- `maintenance-fit-2` pg_amcheck integration dropped without a stated plan. Refuted (no failure scenario): pg_amcheck skips GiST indexes today, so the patch does not create or worsen the scenario. Message #1 shows GIN checking shipped in amcheck without pg_amcheck support, so a function-only step follows existing practice.

### Review angles

- prior-attempts: 1 objection(s). One objection only. The precedent reason that may still apply is the lock-safety question from [P3]. Message #1 links the earlier thread but does not summarize how that question ended.
- standards-compat: 0 objection(s). The user-facing contract follows existing amcheck practice. gist_index_check(index regclass, heapallindexed boolean) matches the naming and argument style of bt_index_check and gin_index_check, and it comes with a normal 1.5 to 1.6 upgrade script. The proposal adds no syntax, GUC, catalog object or pg_dump state, and it does not change behaviour for existing users. Two points could not be checked from the material. First, the documentation hunk shown does not state the lock level, while the bt_index_check entry does; this is a docs gap that belongs to the locking question covered by another angle. Second, pg_amcheck does not support GIN either, so the dropped pg_amcheck part is an existing gap and not a new departure from practice.
- replication-ondisk: 0 objection(s). The proposal adds a read-only contrib function and an amcheck 1.5 to 1.6 upgrade script, with no change to on-disk format, WAL, protocol or core catalogs. Not checkable without code: whether the parent-child recheck is safe on a hot standby during GiST redo, and whether 0003 stops every path that could extend the index.
- security: 0 objection(s). The material shows no new privilege or exposure path: gist_index_check is a read-only amcheck function, and amcheck normally revokes EXECUTE from PUBLIC. Without the code I could not check whether amcheck--1.5--1.6.sql does that, which lock level the function takes, or whether the 0003 crash fixes (P_NEW extension, NULL root) were reviewed.
- maintenance-fit: 2 objection(s). The function lives in contrib/amcheck and adds no cost for users who do not call it. It follows the existing B-tree and GIN checkers, so fit and hot-path cost are not concerns.
- already-exists: 0 objection(s). Postgres has no existing way to verify a GiST index. amcheck covers B-tree, GIN and heap, pageinspect only dumps raw GiST pages for manual reading, and REINDEX replaces the index without reporting corruption. I know of no extension that does this check, and the proposal extends amcheck instead of adding a second mechanism.
- demand: 0 objection(s). No demand objection. amcheck already checks B-tree and GIN, so a GiST check fills a gap in an existing tool. GiST backs core opclasses, exclusion constraints and contrib modules, and several independent reviewers and testers worked on the patch over many CommitFests (messages #3, #6, #8, #12, #16, #17). The cost is a contrib function that runs only when called, so it falls on its users and touches no hot path. The thread cites no field reports of GiST corruption, but a committer would likely accept parity with the existing checkers as enough reason.

### Precedents retrieved

- P1: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P2: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P3: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P4: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant; reason summarised from its thread
- P5: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - used in the review; reason summarised from its thread
- P6: [Speeding up GIST index builds using popcount](https://commitfest.postgresql.org/patch/3023/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [support for box type in SP-GiST index](https://commitfest.postgresql.org/patch/419/) (cf, Returned with feedback, 2016-01-28) - judged not relevant; reason summarised from its thread
- P8: [Better tracking of free space during SP-GiST index build](https://commitfest.postgresql.org/patch/739/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P9: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P10: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant
- P11: [PGSERVICEFILE as part of a normal connection string \[duplicated please delete\]](https://commitfest.postgresql.org/patch/5659/) (cf, Withdrawn, 2025-03-20) - judged not relevant
- P12: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P13: [\[WIP\] Temporal query processing with range types - Temporal Normalization](https://commitfest.postgresql.org/patch/2045/) (cf, Withdrawn, 2020-01-30) - judged not relevant
- P14: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P15: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P16: [Sequence Access Methods](https://commitfest.postgresql.org/patch/466/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P17: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P18: [Reloptions for table access methods](https://commitfest.postgresql.org/patch/2717/) (cf, Withdrawn, 2021-04-06) - judged not relevant
- P19: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [create table like: ACCESS METHOD](https://commitfest.postgresql.org/patch/2865/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P21: [Sequence Access Method](https://commitfest.postgresql.org/patch/44/) (cf, Returned with feedback, 2015-07-28) - judged not relevant
- P22: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P23: [Revert "TupleHashTable: store additional data along with tuple."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4a07f532b40a64fb4714a3f7ab6063435411edb) (revert, Reverted, 2025-01-13) - judged not relevant
- P24: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 119 s; cost reported by the CLI: $1.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 66 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:20.083Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
