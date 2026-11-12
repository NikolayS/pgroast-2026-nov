# Design pre-review: Planner chooses an index-only scan for an index AM without amcanreturn, and execution fails

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7410](https://commitfest.postgresql.org/patch/7410/); [thread](https://www.postgresql.org/message-id/flat/19638-277d0f73dfaeaec8%40postgresql.org), 8 of 8 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducer, including an in-core test module on master (message #3), and the fix is one extra guard in check_index_only() that reuses the existing per-column canreturn flags. The remaining points are a missing regression test and an unsettled list of affected branches, which are not level 1-2 matters. Confidence is medium because the v3 diff and commit message are not shown and no committer has confirmed the analysis.

Not checked: The v3 diff and commit message were not shown, so the exact condition and its effect on indexes with a mix of returnable and non-returnable columns could not be checked. The claim that v3 keeps expression-index and plain-column index-only scans rests on the reporter's tests alone.

## Prior attempts

- [Fix index-only scan plans, take 2.](https://www.postgresql.org/message-id/flat/3179992.1641150853@sss.pgh.pa.us) (related; Partially reverted, 2022-01-03). "Moreover, it broke EXPLAIN for such indexquals (which indicates a gap in our test cases :-()." Different now: P1 changed how IndexOnlyScan rechecks quals on non-returnable columns. This proposal only closes the empty attrs_used case in check_index_only(). P1's remark about weak test coverage is relevant, because make check did not catch the v1 expression-index regression (message #6).

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches fail with the in-core test module, given that message #1 says 18.6 works and message #4 reports the same errors on 18.6?
2. Which branches should the fix be back-patched to?
3. Will the patch add a test module with an AM that has amoptionalkey = true, amgettuple set and no amcanreturn, so the case stays covered?
4. Does a bisect show that the path-generation mask commit of 2026-01-28 made the failure reachable in the pgvectorscale reproducer?
5. Can the v3 patch carry an updated commit message that describes the per-column canreturn check and the expression-index case it preserves?

## Touch points

- **ABI / extension API:** The fix enforces the documented index AM contract for third-party AMs without amcanreturn; no struct or callback changes.
- **planner behaviour change:** check_index_only() in indxpath.c stops building index-only scan paths for indexes where no column is returnable.
- **backpatchability:** It is a bug fix, and messages #1 and #4 disagree on whether 18.6 is affected, so the back-patch range is open.

## Who would commit it, and when

- Owner: silent. Someone who knows index path generation in the planner and the index AM API. David Rowley replied in message #2 with a mixed-binaries theory and has not replied since the in-core test module was posted.
- Priority: needs champion; audience niche; review cost low. The failure needs a third-party AM with amoptionalkey = true, amgettuple and no amcanreturn, but it is a core planner bug that produces an unexecutable plan. The patch touches one function in indxpath.c, and reporter and author agree on v3, so it now waits for a committer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
