# Design pre-review: Wrong result for whole-row IS NOT NULL with dropped columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7494](https://commitfest.postgresql.org/patch/7494/); [thread](https://www.postgresql.org/message-id/flat/DM9RAXA7RBH4.1O1CGGXT98IDY%40gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The proposal fixes a reproducible wrong result by making plan-time folding match ExecEvalRowNullInt(), and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: indexes and stored values built under the old folding keep the old result after a minor update. A committer who knows the planner still has to sign the back-patch to six branches, two of them untested.

Not checked: The code was not read, so the tuple descriptor lookup and release, and the per-branch API differences, are unchecked. Messages #4 and #5 were not shown.

## Prior attempts

None found before 2026-10-20.

## Objections a reviewer is likely to raise

### 1. Back-patched fix leaves expression and partial indexes built under old folding

A user on REL_16 has a partial index with predicate `ROW(a, b)::ts IS NOT NULL` on a table with a dropped column. Before the fix the predicate folded to false, so the index holds no rows and the planner never chose it. After the minor update the planner proves the predicate and uses the empty index, so queries silently miss every pre-existing row.

Index expressions, index predicates and stored generated columns go through eval_const_expressions() when values are computed, so values written before the fix carry the old false result. The patch changes the folded result on all branches down to 14, but the thread and commit messages do not say what happens to data already derived from that folding. The same applies to an expression index on such a test, where old entries hold false. Back-patched planner fixes of this kind usually come with release-note advice to REINDEX affected indexes, and the proposal does not yet say that.

**Question:** Which stored objects (expression indexes, partial indexes, stored generated columns) can hold values computed by the old folding, and what release-note advice tells users to rebuild them?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/DM9RAXA7RBH4.1O1CGGXT98IDY%40gmail.com), [#4](https://www.postgresql.org/message-id/DMSGUXT2T9MW.2Z13MYYFO2E08%40gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which stored objects (partial indexes, expression indexes, stored generated columns) can hold values computed by the old folding, and what release-note advice tells users to rebuild them?
2. Have the REL_15 and REL_16 patches been built and run through the regression tests?
3. What does the tuple descriptor lookup cost at plan time for a query with many NullTests on named-type RowExprs?
4. Do other planner simplifications on RowExpr, such as predicate proof, treat the dropped-column placeholder as a real null field?
5. Does the stored view case, where the RowExpr keeps a Var for the dropped attribute, need any fix beyond this NullTest change?

## Touch points

- **planner behaviour change:** eval_const_expressions() no longer folds ROW(...)::type IS NOT NULL to false when the rowtype has a dropped column.
- **backpatchability:** Separate patches target REL_14 to REL_18; partial and expression indexes built under the old folding keep old contents.
- **performance on the hot path:** The planner looks up the rowtype's tuple descriptor for each NullTest on a named-type RowExpr; no planning-time measurement is shown.

## Who would commit it, and when

- Owner: silent. Someone who knows planner expression simplification in clauses.c and the handling of dropped attributes in named rowtypes.
- Priority: needs champion; audience niche; review cost low. The bug causes silent row loss but needs a dropped column plus a whole-row or cast-to-rowtype null test. The patch is small and confined to clauses.c and tests, two reviewers found no code concerns, and no committer has replied.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
