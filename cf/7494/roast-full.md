# Design pre-review: Wrong result for whole-row IS NOT NULL with dropped columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7494](https://commitfest.postgresql.org/patch/7494/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [Wrong result for whole-row IS NOT NULL with dropped columns](https://www.postgresql.org/message-id/flat/DM9RAXA7RBH4.1O1CGGXT98IDY%40gmail.com), started 2026-10-20
- Messages read: 5 of 5
- Patch prose read: v2-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch, v2-REL_18-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch.nocfbot, v2-REL_17-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch.nocfbot, v2-REL_16-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch.nocfbot, v2-REL_15-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch.nocfbot, v2-REL_14-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch.nocfbot, v1-0001-Ignore-dropped-columns-when-simplifying-ROW-.-IS-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The proposal fixes a reproducible wrong result by making plan-time folding match ExecEvalRowNullInt(), and no objection survived at levels 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: indexes and stored values built under the old folding keep the old result after a minor update. A committer who knows the planner still has to sign the back-patch to six branches, two of them untested.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the tuple descriptor lookup and release, and the per-branch API differences, are unchecked. Messages #4 and #5 were not shown.

## The proposal, restated

**Problem.** The planner folds ROW(...)::type IS NOT NULL to constant false when the named rowtype has a dropped column, because it treats the dropped-column placeholder as a null field. Users get wrong results: a WHERE clause silently drops every row, and a whole-row CHECK constraint cannot be added or satisfied. Virtual generated columns expose it most easily, but casts to a named rowtype hit it on all supported branches.

**Proposed design.** When eval_const_expressions() simplifies a NullTest on a RowExpr of a named composite type, it looks up the rowtype's tuple descriptor and skips fields for dropped attributes. This matches ExecEvalRowNullInt() in the executor. Back-branch versions down to 14 use the same approach, with small API differences.

**What the author claims.**
- Plan-time and execution-time results differ today; the fix makes the planner match the executor.
- The ROW(...)::type case exists on PG 14 and later, so all supported branches need the fix.
- With the patch, EXPLAIN shows per-field tests such as ((a IS NOT NULL) AND (b IS NOT NULL)) instead of false.
- Checking attisdropped in the tuple descriptor also covers stored views whose RowExpr still holds a Var for the dropped attribute.

**Thread state.** Two reviewers, Sho Ito and Joao Rito, report no remaining code concerns; Rito tested v2 on master, REL_18, REL_17 and REL_14. No committer has replied, and the REL_15 and REL_16 patches are untested.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-20; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patched fix leaves expression and partial indexes built under old folding

fixable · level 2, rubric item 13 · backpatchability, planner behaviour change · `standards-compat-1`

A user on REL_16 has a partial index with predicate `ROW(a, b)::ts IS NOT NULL` on a table with a dropped column. Before the fix the predicate folded to false, so the index holds no rows and the planner never chose it. After the minor update the planner proves the predicate and uses the empty index, so queries silently miss every pre-existing row.

Index expressions, index predicates and stored generated columns go through eval_const_expressions() when values are computed, so values written before the fix carry the old false result. The patch changes the folded result on all branches down to 14, but the thread and commit messages do not say what happens to data already derived from that folding. The same applies to an expression index on such a test, where old entries hold false. Back-patched planner fixes of this kind usually come with release-note advice to REINDEX affected indexes, and the proposal does not yet say that.

Thread: [#1 ("Matheus Alcantara", 2026-10-20)](https://www.postgresql.org/message-id/DM9RAXA7RBH4.1O1CGGXT98IDY%40gmail.com), [#4 ("Matheus Alcantara", 2026-11-11)](https://www.postgresql.org/message-id/DMSGUXT2T9MW.2Z13MYYFO2E08%40gmail.com)

**Question:** Which stored objects (expression indexes, partial indexes, stored generated columns) can hold values computed by the old folding, and what release-note advice tells users to rebuild them?

Counter-check: held. Postgres runs index predicates, index expressions and stored generated column expressions through eval_const_expressions() at build and insert time, so old entries reflect the wrong folding. The thread and commit messages do not mention REINDEX advice; the case is narrow and needs only a release-note line.

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
| planner behaviour change | **yes** | eval_const_expressions() no longer folds ROW(...)::type IS NOT NULL to false when the rowtype has a dropped column. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Separate patches target REL_14 to REL_18; partial and expression indexes built under the old folding keep old contents. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The planner looks up the rowtype's tuple descriptor for each NullTest on a named-type RowExpr; no planning-time measurement is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which stored objects (partial indexes, expression indexes, stored generated columns) can hold values computed by the old folding, and what release-note advice tells users to rebuild them?
2. Have the REL_15 and REL_16 patches been built and run through the regression tests?
3. What does the tuple descriptor lookup cost at plan time for a query with many NullTests on named-type RowExprs?
4. Do other planner simplifications on RowExpr, such as predicate proof, treat the dropped-column placeholder as a real null field?
5. Does the stored view case, where the RowExpr keeps a Var for the dropped attribute, need any fix beyond this NullTest change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows planner expression simplification in clauses.c and the handling of dropped attributes in named rowtypes.
- **Liability:** A planner result change back-patched to all supported branches, including any later report of stale partial or expression indexes after the minor update.
- **Author follow-up:** The author posted v2 with patches for REL_14 to REL_18 after review, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug causes silent row loss but needs a dropped column plus a whole-row or cast-to-rowtype null test. The patch is small and confined to clauses.c and tests, two reviewers found no code concerns, and no committer has replied.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P8] also touches NULL folding in eval_const_expressions, but it added new folding based on NOT NULL constraints, while this patch removes a wrong fold so the planner matches ExecEvalRowNullInt(). Its key sentence, about waiting for better infrastructure before adding new NULL smarts, does not apply to a wrong-result fix. The thread cites no earlier discussion of this bug.
- standards-compat: 1 objection(s). Apart from the stored-data question, the change makes plan-time folding match the executor and the SQL meaning of a row without dropped attributes. It adds no syntax, GUC, catalog or pg_dump surface. [P8] concerns adding new NULL folding and does not apply to fixing existing folding.
- replication-ondisk: 0 objection(s). The proposal changes only plan-time folding in clauses.c, with no change to on-disk format, WAL, catalogs, wire protocol or exported structs, so pg_upgrade, standbys and back-branch ABI are unaffected. Stored views and CHECK constraints keep their stored trees and are re-planned with the fixed logic after a minor update.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how the planner folds a NullTest so the result matches what ExecEvalRowNullInt() already returns. It adds no user surface, privilege, catalog access or resource use that a user controls. The fix makes an RLS policy or security barrier view that uses ROW(...) IS NOT NULL give the result the executor already defines, so it opens no new access path.
- maintenance-fit: 0 objection(s). This is a wrong-result fix that brings the planner in line with ExecEvalRowNullInt(), so it adds no new mechanism to maintain. [P8] does not apply: its rejected design opened the table in eval_const_expressions(), and this patch reads the rowtype's tuple descriptor, which costs only planning of NullTests on named RowExprs. No committer has replied yet, and the REL_15 and REL_16 patches are untested. Those points affect committability, not the merits.
- already-exists: 0 objection(s). This is a wrong-result bug fix in eval_const_expressions(), so no existing feature, extension or hook can replace it. Users can only avoid the folding by rewriting queries, and they cannot do that for a whole-row CHECK constraint on a table with virtual generated columns. [P8] concerns adding new NULL folding, and its recorded reason does not apply to removing incorrect folding.
- demand: 0 objection(s). This is a wrong-result bug fix with no new user-visible surface, so the demand test does not apply in the usual way. The thread shows reproducible silent row loss in WHERE clauses and CHECK constraints that can never be satisfied, reached through casts to named rowtypes on all supported branches. Correct query results are needed by every user who hits the case, and the cost falls only on the planner's handling of NullTest on named-type RowExprs. [P8] is about adding new NULL-folding smarts, not about fixing an existing incorrect fold, so it does not apply.

### Precedents retrieved

- P1: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P2: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P3: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P4: [Rename constants in preparation of variableness](https://commitfest.postgresql.org/patch/4828/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P5: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P6: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P7: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant; reason summarised from its thread
- P8: [Optimizing NULL check on pkey column](https://commitfest.postgresql.org/patch/2699/) (cf, Returned with feedback, 2021-09-27) - used in the review; reason summarised from its thread
- P9: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P10: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P11: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P12: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P13: [JIT expression evaluation improvements](https://commitfest.postgresql.org/patch/2501/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P14: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P15: [\[PATCH\]Refactor and unify expression construction functions in makefuncs.c](https://commitfest.postgresql.org/patch/6823/) (cf, Returned with feedback, 2026-06-01) - judged not relevant
- P16: [Allow placeholders in ALTER ROLE w/o superuser](https://commitfest.postgresql.org/patch/3749/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P18: [Revert "Do execGrouping.c via expression eval machinery."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2a41507dab0f293ff241fe8ae326065998668af8) (revert, Reverted, 2018-02-16) - judged not relevant
- P19: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P20: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P21: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P22: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant
- P23: [Check for tuplestorestate nullness before dereferencing](https://commitfest.postgresql.org/patch/5705/) (cf, Withdrawn, 2025-12-08) - judged not relevant
- P24: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 81 s; cost reported by the CLI: $1.38
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:00:44.148Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
