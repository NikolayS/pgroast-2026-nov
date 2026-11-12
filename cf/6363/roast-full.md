# Design pre-review: Add a berief general comment on BTScanInsertData's nextkey and backward

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6363](https://commitfest.postgresql.org/patch/6363/) (Needs review); authors: Yugo Nagata (yugo.nagata)
- Thread: [Add a berief general comment on BTScanInsertData's nextkey and backward](https://www.postgresql.org/message-id/flat/20251118162801.27cb13408cb3c9dd3a72cf7d%40sraoss.co.jp), started 2025-11-18
- Messages read: 4 of 4
- Patch prose read: v2-0002-Add-a-berief-general-comment-on-BTScanInsertData-.patch, v2-0001-Clarify-_bt_binsrch-comments-for-internal-vs-leaf.patch, add_brief_general_comment_on_BTScanInsertData_nextkey.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patches change comments only in nbtree.h and nbtsearch.c, and Peter Geoghegan called the first comment reasonable in message #2. The remaining risk is whether the v2-0001 wording is accurate, and the material does not show that text.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The new comment text in v2-0001 and v2-0002 was not read, so its accuracy against _bt_binsrch and _bt_first could not be checked.

## The proposal, restated

**Problem.** The comment on BTScanInsertData no longer explains the nextkey and backward fields and only points to _bt_first. The author thinks commit c9c0589fda0e removed the old text when the backward scan optimization changed their meaning. Readers of the nbtree code are the audience.

**Proposed design.** Patch 0002 adds a short comment in nbtree.h: nextkey sets how the scankey boundary is read, backward marks a backward scan, and _bt_first has the details. Patch 0001 rewrites the _bt_binsrch comments in nbtsearch.c to separate internal page behavior from leaf page behavior, after a reviewer's remark. The patches change comments only.

**What the author claims.**
- A brief, general description of nextkey and backward in BTScanInsertData helps readers.
- The _bt_binsrch comment and the end of _bt_first do not contradict each other, but naming the leaf page would make this clearer.
- The follow-up patch makes the difference between internal and leaf page behavior in _bt_binsrch clearer.

**Thread state.** Peter Geoghegan called the first comment reasonable and asked whether the _bt_binsrch internal page comments, which seem to contradict _bt_first, should also change. The author posted v2 with a follow-up patch in March 2026 and asked if it helps; nobody has replied since.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-11-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | New comments in nbtree.h and nbtsearch.c must stay accurate when _bt_first or _bt_binsrch change. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the new _bt_binsrch comment in v2-0001 use the pivot tuple and downlink explanation from message #2, or a different one?
2. Is the suggested edit to the comment at the end of _bt_first, from message #3, part of v2?
3. Does v2-0002 change the nbtree.h wording from the first version?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Geoghegan replied in message #2 and asked for the _bt_binsrch change; a committer who knows nbtree search must confirm the v2-0001 wording.
- **Liability:** The committer signs for the accuracy of two comments on nbtree search. A wrong comment misleads later readers, but no behavior changes.
- **Author follow-up:** The author posted v2 after the review and moved the entry through three CommitFests, most recently on 2026-10-23.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Two small comment-only patches help readers of the nbtree code and add no runtime cost. The entry has waited since March 2026 for a reply to v2 from someone who knows nbtree.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: [P5] and [P3] are comment-only patches in other subsystems, and neither recorded reason (a wrong walsender state; no author reply to review) applies here. The thread cites no earlier attempt beyond commit c9c0589fda0e (message #1), which removed the old text and is not a rejected proposal.
- standards-compat: 0 objection(s). The patches change only source comments in nbtree.h and nbtsearch.c. Nothing changes in SQL syntax, GUCs, pg_dump output, psql or what clients see, so the standards and compatibility angle has nothing to object to. Whether the new _bt_binsrch wording is accurate is a separate question, and the material does not show that text.
- replication-ondisk: 0 objection(s). The patches change comments only, in nbtree.h and nbtsearch.c. They change no on-disk format, WAL record, catalog, wire protocol or struct layout, so this angle has nothing to object to. The nbtree VACUUM bookkeeping invariant does not apply because no VACUUM behavior changes.
- security: 0 objection(s). The patches change only comments in nbtree.h and nbtsearch.c. They add no privilege, exposure, authentication or resource-use aspect, so the security angle has nothing to object to.
- maintenance-fit: 0 objection(s). The patches change comments only, so non-users pay no runtime, ABI or upgrade cost, and the nbtree VACUUM bookkeeping invariant does not apply. Peter Geoghegan, who knows this area, called the first comment reasonable in message #2, and v2-0001 answers his own request; the v2-0001 wording is not in the material, so its accuracy could not be checked.
- already-exists: 0 objection(s). The patches change comments only, so no existing feature, extension or tool can meet this need. The existing pointer to _bt_first is the current way to learn what the fields mean, and the proposal only adds a short summary next to it, which is not a second mechanism.
- demand: 0 objection(s). The patch changes comments only, so the demand test does not apply in the usual way. It adds no user surface and puts no cost on non-users. The readers are nbtree code readers, and Peter Geoghegan called the first comment reasonable in message #2. Whether the v2 wording is correct is a separate question for other angles.

### Precedents retrieved

- P1: [Using indexUnchanged with nbtree](https://commitfest.postgresql.org/patch/3191/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P2: [Revert "nbtree: Remove useless row compare arg."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd2ce3792754e0705884aed15d1fd2ae43cc1500) (revert, Reverted, 2025-06-11) - judged not relevant
- P3: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - used in the review; reason summarised from its thread
- P4: [revert: Remove meaningless assignments in nbtree code](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b93e9a5c94b4c89932a637798bd560971fe790d7) (revert, Reverted, 2019-12-19) - judged not relevant
- P5: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - used in the review; reason summarised from its thread
- P6: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P7: [Support to COMMENT ON DATABASE CURRENT_DATABASE](https://commitfest.postgresql.org/patch/1320/) (cf, Rejected, 2018-03-06) - judged not relevant; reason summarised from its thread
- P8: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P9: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P10: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant; reason summarised from its thread
- P11: [Delaying/avoiding BTreeTupleGetNAtts() call within _bt_compare()](https://commitfest.postgresql.org/patch/2429/) (cf, Withdrawn, 2020-11-09) - judged not relevant
- P12: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P13: [Support to COMMENT ON CURRENT DATABASE](https://commitfest.postgresql.org/patch/229/) (cf, Returned with feedback, 2015-09-03) - judged not relevant
- P14: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [dual pivot quicksort](https://commitfest.postgresql.org/patch/3738/) (cf, Withdrawn, 2022-07-22) - judged not relevant
- P16: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P17: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P20: [Clarify ItemPointers for Table AM API](https://commitfest.postgresql.org/patch/3108/) (cf, Withdrawn, 2021-06-18) - judged not relevant
- P21: [Clarify need for memory barriers in bgworkers](https://commitfest.postgresql.org/patch/69/) (cf, Rejected, 2015-02-03) - judged not relevant
- P22: [KNN-btree](https://commitfest.postgresql.org/patch/966/) (cf, Returned with feedback, 2017-03-09) - judged not relevant
- P23: [revert: add transaction processing chapter with internals info](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4dd00b042fe88bfbf7ce6400c69ee232e9eb6e48) (revert, Reverted, 2022-12-01) - judged not relevant
- P24: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 71 s; cost reported by the CLI: $0.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 39 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:49:14.416Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
