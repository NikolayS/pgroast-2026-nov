# Design pre-review: Use-after-free of a shared Path in add_path()/add_partial_path()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7498](https://commitfest.postgresql.org/patch/7498/); [thread](https://www.postgresql.org/message-id/flat/CAM2%2B6%3DXaYtMGwh27O7oDTfAw7kV4VdUVbxC2RVzWtKz_ysekKw%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with two reproducers, Richard Guo confirmed the bug on v19 and master in message #3, and the fix changes no signature or user-visible behaviour. The remaining points are the choice among the parent_rel check, a flat copy and reference counting, and the back-patch range, which a planner committer has to settle.

Not checked: Without the code, nobody here could check that every path constructor sets path->parent as the fix assumes, or whether the patch updates the add_path() header comment and optimizer/README. The create_ordered_paths() case has no regression test, so its coverage rests on the author's manual check.

## Prior attempts

None found before 2026-09-09.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which create_*_path() constructors, including FDW and custom scan paths, were checked to set path->parent to the rel later passed to add_path()?
2. Can a path be shared across rels while path->parent equals parent_rel, so that the check still frees it?
3. How far back should the fix go, and does Richard Guo's v19 reproducer also fail on the older supported branches?
4. Why is the parent_rel check preferred over a flat copy of the path at the two call sites, which Richard Guo says also fixes the bug?
5. How much extra planner memory do the unfreed paths cost on a query with many partitions or many postgres_fdw upper paths?

## Touch points

- **ABI / extension API:** add_path() and add_partial_path() keep their signatures, but FDW and custom paths whose parent differs from parent_rel are no longer freed.
- **planner behaviour change:** The patch changes when add_path() and add_partial_path() free a dominated or rejected path; plan choice is meant to stay the same.
- **backpatchability:** The bug is on all branches and the author asks for a back-patch; no reviewer has said how far back.
- **maintenance burden ("who maintains this in 10 years"):** The fix relies on every path constructor setting path->parent, a rule future path types must keep.
- **performance on the hot path:** add_path() runs for every candidate path; the patch adds one pointer compare and keeps some paths until planning ends.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the planner's path generation and upper-rel code. Richard Guo reviewed in message #3; the visible thread does not show whether he will commit it.
- Priority: proceed; audience a sizeable minority; review cost low. This is a planner failure reachable from plain SQL under parallel query, reproduced by a reviewer on v19 and master. The patch is about 7 KiB in two planner files plus tests, with no API change.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
