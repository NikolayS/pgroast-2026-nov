# Design pre-review: amcheck: support for GiST

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5879](https://commitfest.postgresql.org/patch/5879/); [thread](https://www.postgresql.org/message-id/flat/5FC1B5B6-FB35-44A2-AB62-632F14E958C5%40yandex-team.ru), 17 of 17 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The goal fits amcheck and no objection survived at levels 1a or 1b. One fixable level 2 objection survived, prior-attempts-1: the visible thread does not state the relation lock level or explain why the parent and child recheck is correct under concurrent page deletion, the two points recorded in P3. A committer who knows GiST can settle this from a short written answer and the code.

Not checked: Without the code, the lock level, the recheck under concurrent page deletion, behaviour on a standby and the 0003 fixes could not be checked. Only the first 3 of 17 messages were read in full.

## Prior attempts

- [amcheck verification for GiST](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru) (same idea; Returned with feedback, 2019-11-28). "My main concern now is the heavyweight lock strength needed by the new function." ([message](https://www.postgresql.org/message-id/CAH2-WznzhHKREeXs1RMFYes%2BAo7aam5dPSmvS2Hto6WUcjEk4A%40mail.gmail.com)) Different now: The function is rebased, with shared tuple normalization for heapallindexed, extra corruption checks and fixes for false positives. The visible thread does not say what changed about lock strength or page deletion.
- [Verify true root on replicas with amcheck](https://www.postgresql.org/message-id/flat/AAF5CE5C-5152-44D8-B437-A0784697DBA5@gmail.com) (related; Returned with feedback, 2020-04-08). "Your patch looks like it gets it right (it won't deadlock with other sessions that access the metapage), but I hesitate to commit it without a strong justification." ([message](https://www.postgresql.org/message-id/CAH2-WzkxnA39Rzad%2BM7DJOQ3Up%3D_7dTE-6jXmx%2B4mp%2B5v%2BKWzw%40mail.gmail.com)) Different now: P5 was a B-tree true-root check on replicas. It is related only because that check also held two buffer locks at once and was asked for a strong justification.

## Objections a reviewer is likely to raise

### 1. Lock strength and page-deletion concerns from the 2019 attempt are unanswered

A DBA runs gist_index_check on a busy production table while VACUUM deletes empty GiST leaf pages. The check follows a downlink to a deleted or reused page, or re-finds the wrong parent, and reports corruption that is not there. The DBA then rebuilds a healthy index or opens a false bug report.

[P3] is the same function by the same author, returned in 2019. The recorded concern there was the heavyweight lock level (AccessShareLock against an earlier ShareLock), whether the parent re-find logic copied from gistfinishsplit() holds here, and concurrent page deletion. None of the 17 messages states the lock level or explains why the re-check is safe against page deletion. Message #4 touches concurrency only for downlink bounds, and the commit message only says that locks follow page-split order. [P5] shows that holding several buffer locks at once was also asked to carry a strong justification.

Precedent: [amcheck verification for GiST](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru) (Returned with feedback, 2019-11-28)

Precedent: [Verify true root on replicas with amcheck](https://www.postgresql.org/message-id/flat/AAF5CE5C-5152-44D8-B437-A0784697DBA5@gmail.com) (Returned with feedback, 2020-04-08)

**Question:** Which heavyweight lock does gist_index_check take, and why is the parent and child re-check correct under that lock when VACUUM deletes GiST pages concurrently?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/5FC1B5B6-FB35-44A2-AB62-632F14E958C5%40yandex-team.ru), [#4](https://www.postgresql.org/message-id/CALdSSPhs4sCd5S_euKxTufk8sOA739ydBwhoGFYQenya7YZyiA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/MEAPR01MB30311C5AE2220142641DA0C2B683A%40MEAPR01MB3031.ausprd01.prod.outlook.com); `prior-attempts-1`</sub>

## Questions for the author

1. Which heavyweight lock does gist_index_check take on the index and on the table, and does it block writes?
2. Why is the parent and child recheck correct when VACUUM deletes or recycles GiST pages during the check?
3. Does the parent re-find logic questioned in P3 still exist in the current patch, and if so, why does it hold here?
4. Is the pg_amcheck part dropped, moved to another thread, or still planned for this series?
5. Has anyone reviewed the 0003 fixes, including the downlink block-number checks from message #3 and the TAP test that corrupts the index file?

## Touch points

- **ABI / extension API:** amcheck moves to version 1.6 with amcheck--1.5--1.6.sql, and tuple normalization moves from verify_nbtree.c into verify_common.
- **security / privileges:** A new SQL function reads any GiST index and its heap; the upgrade script must set EXECUTE privileges as other amcheck functions do.
- **maintenance burden ("who maintains this in 10 years"):** verify_gist.c takes parent and child locks in page-split order, so it must follow future changes to GiST split and page deletion.

## Who would commit it, and when

- Owner: silent. Someone who knows GiST page split and page deletion locking, and the amcheck conventions for lock levels and false positives. The latest message asks for a committer and none has replied.
- Priority: needs champion; audience a sizeable minority; review cost medium. The check fills a gap in amcheck for a widely used index type and costs nothing for users who do not call it. The entry is Ready for Committer after several reviews, but no committer has replied, and the lock-level question from P3 needs a written answer first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
