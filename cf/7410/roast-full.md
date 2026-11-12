# Design pre-review: Planner chooses an index-only scan for an index AM without amcanreturn, and execution fails

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7410](https://commitfest.postgresql.org/patch/7410/) (Needs review); authors: Andrey Rachitskiy (a.rachitskiy)
- Thread: [BUG #19638: Planner chooses an index-only scan for an index AM without amcanreturn, and execution fails](https://www.postgresql.org/message-id/flat/19638-277d0f73dfaeaec8%40postgresql.org), started 2026-08-23
- Messages read: 8 of 8
- Patch prose read: 0002-Reject-index-only-scans-when-index-cannot-return.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducer, including an in-core test module on master (message #3), and the fix is one extra guard in check_index_only() that reuses the existing per-column canreturn flags. The remaining points are a missing regression test and an unsettled list of affected branches, which are not level 1-2 matters. Confidence is medium because the v3 diff and commit message are not shown and no committer has confirmed the analysis.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v3 diff and commit message were not shown, so the exact condition and its effect on indexes with a mix of returnable and non-returnable columns could not be checked. The claim that v3 keeps expression-index and plain-column index-only scans rests on the reporter's tests alone.

## The proposal, restated

**Problem.** The planner builds an index-only scan over an index AM that has amoptionalkey = true, amgettuple set and no amcanreturn, and the executor then fails with "no data returned for index-only scan". It affects queries whose target list needs no column of the table, such as count(*), select with an empty list and EXISTS. The report shows the failure on 19beta3 with the third-party diskann AM, and 18.6 plans a disabled seq scan for the same query.

**Proposed design.** check_index_only() gets one more guard after the bms_is_subset() test, which is true when attrs_used is empty. The first patch rejected the path when index_canreturn_attrs is empty. That also rejected valid index-only scans over expression-only indexes, so v3 checks the per-column canreturn flags inside the existing loop instead.

**User-visible surface.**
- Planner behaviour only: index-only scans are no longer chosen for indexes where no column is returnable.

**What the author claims.**
- The fix is minimal and touches only check_index_only() in indxpath.c.
- The reporter says v3 fixes the reproducer and the select-with-empty-list and EXISTS cases on 18.6.
- The reporter says v3 keeps index-only scans for expression indexes and plain column indexes.
- make check passes, 231 of 231 tests, on 18.6 with v3.
- The same class of problem was fixed in get_actual_variable_range() by 74197bdc842.

**Thread state.** The reporter and Andrey agree on v3, and the reporter has nothing further. David Rowley first suspected mixed binaries and asked how pgvectorscale was built for 19; he has not replied since the in-core test module was posted.

## 2. Prior attempts

### Fix index-only scan plans, take 2. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c)
- **Thread:** [https://www.postgresql.org/message-id/flat/3179992.1641150853@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/3179992.1641150853@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/17350-b5bdcf476e5badbb@postgresql.org](https://www.postgresql.org/message-id/flat/17350-b5bdcf476e5badbb@postgresql.org)
- **Date:** decided 2022-01-03; started 2022-01-01
- **Outcome:** Partially reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Commit 4ace45677 failed to fix the problem fully, because the same issue of attempting to fetch a non-returnable index column can occur when rechecking the indexqual after using a lossy index operator. Moreover, it broke EXPLAIN for such indexquals (which indicates a gap in our test cases :-(). Revert the code changes of 4ace45677 in favor of adding a new field to struct IndexOnlyScan, containing a version of the indexqual that can be executed against the index-returned tuple without using any non-returnable columns. (The restrictions imposed by check_index_only guarantee this is possible, although we may have to recompute indexed expressions.) Support construction of that during setrefs.c processing by marking IndexOnlyScan.indextlist entries as resjunk if they can't be returned, rather than removing them entirely. (We could alternatively require setrefs.c to look up the IndexOptInfo again, but abusing resjunk this way seems like a reasonably safe way to avoid needing to do that.) This solution isn't great from an API-stability standpoint: if there are any extensions out there that build IndexOnlyScan structs directly, they'll be broken in the next minor releases. However, only a very invasive extension would be likely to do such a thing. There's no change in the Path representation, so typical planner extensions shouldn't have a problem. As before, back-patch to all supported branches.
- **Reason (quote):** "Moreover, it broke EXPLAIN for such indexquals (which indicates a gap in our test cases :-()."
- **What is different this time:** P1 changed how IndexOnlyScan rechecks quals on non-returnable columns. This proposal only closes the empty attrs_used case in check_index_only(). P1's remark about weak test coverage is relevant, because make check did not catch the v1 expression-index regression (message #6).


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The fix enforces the documented index AM contract for third-party AMs without amcanreturn; no struct or callback changes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | check_index_only() in indxpath.c stops building index-only scan paths for indexes where no column is returnable. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | It is a bug fix, and messages #1 and #4 disagree on whether 18.6 is affected, so the back-patch range is open. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches fail with the in-core test module, given that message #1 says 18.6 works and message #4 reports the same errors on 18.6?
2. Which branches should the fix be back-patched to?
3. Will the patch add a test module with an AM that has amoptionalkey = true, amgettuple set and no amcanreturn, so the case stays covered?
4. Does a bisect show that the path-generation mask commit of 2026-01-28 made the failure reachable in the pgvectorscale reproducer?
5. Can the v3 patch carry an updated commit message that describes the per-column canreturn check and the expression-index case it preserves?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows index path generation in the planner and the index AM API. David Rowley replied in message #2 with a mixed-binaries theory and has not replied since the in-core test module was posted.
- **Liability:** A planner check that decides when index-only scans are allowed, back-patched to an unsettled set of branches. A wrong condition would silently remove valid index-only scans, as v1 did for expression-only indexes.
- **Author follow-up:** Good sign: the author produced v3 after the reporter found that v1 rejected valid index-only scans over expression-only indexes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The failure needs a third-party AM with amoptionalkey = true, amgettuple and no amcanreturn, but it is a core planner bug that produces an unexecutable plan. The patch touches one function in indxpath.c, and reporter and author agree on v3, so it now waits for a committer.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a reason that applies here. P1 is a different fix in the same area, and its reason does not hit v3, because any qual column makes attrs_used non-empty and the existing subset test rejects the path. P3 concerned bugs in a different feature, so it is not listed. The thread cites one earlier discussion, the get_actual_variable_range() check (messages #1 and #3), and describes it as a fix for the same assumption in another place, not as a rejection.
- standards-compat: 0 objection(s). The change affects only queries that fail today with "no data returned for index-only scan", and v3 keeps index-only scans for expression and plain column indexes (messages #6 and #8). It adds no syntax, GUC, catalog or dump change, so a back-patched plan change has no working query to break.
- replication-ondisk: 0 objection(s). The fix changes only the path test in check_index_only(). It reads the existing IndexOptInfo canreturn flags and adds no on-disk, WAL, catalog, wire protocol or struct change. Plans are not persistent, so standbys, pg_upgrade and back-patching across minor releases are unaffected, and extensions see no ABI change.
- security: 0 objection(s). The proposal only stops the planner from building index-only scan paths it cannot execute. No privilege, role, visibility rule, RLS policy or new information channel is added, and an unprivileged user gains nothing new, so the security angle finds no objection.
- maintenance-fit: 0 objection(s). No maintenance or fit objection. v3 adds one flag inside an existing per-index loop in check_index_only(), so planning cost is unchanged, there is no new API, GUC or catalog state, and the fix closes a gap in the documented index AM contract. Not checked without the code: the missing regression test module is a level 4 matter, and a committer has not engaged since message #2.
- already-exists: 0 objection(s). From the already-exists angle there is nothing to raise. This is a planner bug fix, and no existing feature, setting or extension stops the planner from choosing an index-only scan over an index AM that cannot return columns. v3 reuses the existing per-column canreturn[] flags filled from index_can_return() and adds no second mechanism.
- demand: 0 objection(s). Demand is shown. The planner builds an unexecutable plan for an index AM configuration that the AM API allows, and the diskann AM in pgvectorscale, a third-party extension in real use, exposes it through count(*), select with an empty target list and EXISTS. The extension cannot work around this, because an amcanreturn that always returns false leaves the planner in the same state as a NULL one.

### Precedents retrieved

- P1: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - used in the review
- P2: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P3: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - used in the review; reason summarised from its thread
- P4: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P5: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P6: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P7: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P8: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P10: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P11: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant; reason summarised from its thread
- P12: [Add minimal C example and SQL registration example for custom table access methods](https://commitfest.postgresql.org/patch/4973/) (cf, Withdrawn, 2024-07-25) - judged not relevant
- P13: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P14: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/4617/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P15: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/1237/) (cf, Returned with feedback, 2017-09-19) - judged not relevant
- P16: [Revert: Allow table AM tuple_insert() method to return the different slot](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8dd0bb84da7d56a9e41241b26bfbf6b79644d574) (revert, Reverted, 2024-04-11) - judged not relevant
- P17: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P18: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P19: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P20: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P22: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant
- P23: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P24: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 60 s; cost reported by the CLI: $0.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:52:42.468Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
