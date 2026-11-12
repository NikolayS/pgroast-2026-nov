# Design pre-review: Clarify wait versus poll behavior in ExecAppendAsyncEventWait

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7612](https://commitfest.postgresql.org/patch/7612/) (Needs review); authors: Chao Li (chaol)
- Thread: [Clarify wait versus poll behavior in ExecAppendAsyncEventWait](https://www.postgresql.org/message-id/flat/C05FA422-F3E6-4198-AB7F-70570C157001%40gmail.com), started 2026-11-05
- Messages read: 6 of 6
- Patch prose read: v1-0001-Clarify-when-ExecAppendAsyncEventWait-waits-or-po.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at any level. The patch changes comments only in nodeAppend.c, and Etsuro Fujita, who reviewed v1 in message #2, rewrote it as v2 and, per the brief, pushed it on 2026-11-10. Confidence is medium because the final v2 comment text and the commit are not in the material.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 comment text and the pushed commit are not in the material, so the accuracy of the final wording could not be checked.

## The proposal, restated

**Problem.** The header comment of ExecAppendAsyncEventWait in nodeAppend.c does not say when the function waits for events and when it only polls. This came up in review of another patch, and the reviewer, Etsuro Fujita, asked for the comment change as a separate patch. The audience is developers reading the async Append executor code.

**Proposed design.** v1 adds to the header comment that the function waits when node->as_syncdone is true and polls without waiting otherwise. Etsuro Fujita's v2 rewrites that text using an existing in-function comment about timeout -1 versus 0. v2 then replaces that in-function comment with a brief one. The patch changes comments only.

**What the author claims.**
- The header comment of ExecAppendAsyncEventWait can be improved.
- The reviewer says v1 is not fully correct, because async execution still runs during polling when some events are ready.
- The reviewer says the existing timeout comment is more accurate, and that v2 makes it redundant.

**Thread state.** Etsuro Fujita reviewed v1, posted v2, and pushed it on 2026-11-10 after Chao Li agreed. The CommitFest entry still shows Needs review and Patch needs rebase, so its status has not caught up with the push.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-05; 24 candidates retrieved, none is the same or a similar idea).

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
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which commit and which branches received the v2 comment change?
2. Can the CommitFest entry be marked Committed, since it still shows Needs review and Patch needs rebase?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Etsuro Fujita reviewed v1, wrote v2 in message #2, and pushed it per the brief. The owner needs to know the async Append executor code.
- **Liability:** The accuracy of one header comment and one short in-function comment in nodeAppend.c. There is no runtime behaviour to own.
- **Author follow-up:** Chao Li replied the same day in message #3 and accepted v2. Nothing further is needed from the author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change helps developers who read the async Append code and costs nothing at run time. The brief says it was pushed on 2026-11-10, so the only remaining step is to update the CommitFest entry.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the thread points to no earlier attempt at this comment change. Message #1 links only the patch whose review led to this one. The proposal changes comments only, and the committer who knows the area rewrote and pushed it (messages #2 and #5), so no prior-rejection reason applies.
- standards-compat: 0 objection(s). The patch changes one comment in nodeAppend.c and nothing user-facing. It adds no SQL syntax, GUC, catalog object, psql behaviour, pg_dump output or client-visible change, so the standards and compatibility angle has nothing to object to. A committer who knows the area reviewed it, rewrote it and pushed it.
- replication-ondisk: 0 objection(s). The patch changes only comments in src/backend/executor/nodeAppend.c. It adds no on-disk, WAL, catalog, wire protocol or ABI change, so there is nothing to object to from the replication and on-disk angle.
- security: 0 objection(s). The patch changes only comments in nodeAppend.c. It adds no privileges, defaults, user-visible surface, or runtime behaviour, so there is nothing to object to on security grounds.
- maintenance-fit: 0 objection(s). The patch changes comments in nodeAppend.c only. It adds no hot-path cost, no API, no dependency and no long-term commitment. The committer who reviewed it, Etsuro Fujita, knows the async Append code, rewrote it as v2 and pushed it (message #5). From the maintenance and fit angle there is nothing to object to.
- already-exists: 0 objection(s). The patch only changes comments in nodeAppend.c, and no extension or existing feature can stand in for a code comment. The angle also covers duplication: v2 removes the overlapping in-function comment, so the patch leaves less repeated text. The patch was pushed on 2026-11-10.
- demand: 0 objection(s). This is a comment-only change in nodeAppend.c, so the demand test does not apply in the usual way. Readers of the async Append code gain, nobody pays, and the area committer asked for it, rewrote it, and pushed it in message #5.

### Precedents retrieved

- P1: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - judged not relevant; reason summarised from its thread
- P2: [Parallel Append implementation](https://commitfest.postgresql.org/patch/929/) (cf, Returned with feedback, 2017-02-01) - judged not relevant; reason summarised from its thread
- P3: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P4: [Make async slave to wait for lsn to be replayed](https://commitfest.postgresql.org/patch/772/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [Improvements for pg_wal_replay_wait()](https://commitfest.postgresql.org/patch/5167/) (cf, Returned with feedback, 2024-12-10) - judged not relevant; reason summarised from its thread
- P6: [Make Append Cost aware of some run time partition prune case](https://commitfest.postgresql.org/patch/2829/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P7: [Adjust tuples estimate for appendrel](https://commitfest.postgresql.org/patch/4711/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [doc: Add some OID/TOAST-related limitations to the limits appendix.](https://commitfest.postgresql.org/patch/4320/) (cf, Withdrawn, 2023-06-13) - judged not relevant; reason summarised from its thread
- P9: [Improve Append/MergeAppend EXPLAIN output when all subplans are pruned.](https://commitfest.postgresql.org/patch/2102/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P10: [Clarify the comments about varlena header encoding](https://commitfest.postgresql.org/patch/3820/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P11: [Add new wait event to XactLockTableWait](https://commitfest.postgresql.org/patch/5804/) (cf, Withdrawn, 2026-03-31) - judged not relevant
- P12: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P13: [pgbench --continue-on-error: clarify TPS and failure reporting](https://commitfest.postgresql.org/patch/6942/) (cf, Withdrawn, 2026-06-28) - judged not relevant
- P14: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P15: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P16: [Clarify ItemPointers for Table AM API](https://commitfest.postgresql.org/patch/3108/) (cf, Withdrawn, 2021-06-18) - judged not relevant
- P17: [Support to COMMENT ON DATABASE CURRENT_DATABASE](https://commitfest.postgresql.org/patch/1320/) (cf, Rejected, 2018-03-06) - judged not relevant
- P18: [psql: --echo-hidden improvements](https://commitfest.postgresql.org/patch/4903/) (cf, Withdrawn, 2024-03-21) - judged not relevant
- P19: [Clarify need for memory barriers in bgworkers](https://commitfest.postgresql.org/patch/69/) (cf, Rejected, 2015-02-03) - judged not relevant
- P20: [Revert wrong SCO OpenServer report, update comments and improve formatting a bit. Regenerate INSTALL.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=248365f2c64fb8ce94c28c680d95a279dd07953a) (revert, Reverted, 2001-04-06) - judged not relevant
- P21: [\[PATCH\] Fix typos in pqsignal.c comment](https://commitfest.postgresql.org/patch/6871/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P22: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P23: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P24: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 88 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 38 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:07:10.092Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
