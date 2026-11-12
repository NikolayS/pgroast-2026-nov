# Design pre-review: Fix grouping-set nulling of variable-free grouping expressions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7438](https://commitfest.postgresql.org/patch/7438/) (Needs review); authors: Haibo Yan (billyean)
- Thread: [\[PATCH\] Fix grouping-set nulling of variable-free grouping expressions](https://www.postgresql.org/message-id/flat/CABXr29FSih8YtLCEe6Swquu_c%3DYvFiLVoH-ejvk2jVge9EMEMQ%40mail.gmail.com), started 2026-10-10
- Messages read: 8 of 8
- Patch prose read: v3-0001-Fix-grouping-set-nulling-of-variable-free-grouping-exprs.patch, v1-0001-Fix-grouping-set-nulling-of-variable-free-grouping-exprs.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The bug is real and a fix is wanted, but the proposal now holds two definitions of the correct result (standards-compat-2). The commit message says `(1 + 1) * 10 ... rollup (1 + 1)` and now() cases are wrong results. The parser rules planned in message #6 stop matching such expressions, and message #8 accepts the unmatched result for now(). No posted patch contains those rules, so the list has to agree on the definition before design review.

First failing level, from the base up: **Level 1a: understandability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: failed
- Level 1b: desirability: not evaluated
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The parser rules exist only as a description in the thread, so their effect on currently correct queries and on stored views could not be judged. The planner code in var.c was not read.

## The proposal, restated

**Problem.** Grouping sets return non-null values in rows where a variable-free grouping expression should be NULL, for example `(1 + 1) * 10 ... GROUP BY ROLLUP (1 + 1)` returns 20 in the grand-total row. Postgres 18 already has this bug. In 19, commit e2debb64380 folds `a IS NOT NULL` to true for NOT NULL columns, so common queries now give results that differ from 18.

**Proposed design.** mark_nullable_by_grouping() now reuses one PlaceHolderVar ID per RTE_GROUP column, cached in PlannerInfo, instead of making a new ID for each reference. Then setrefs matches every reference to the grouping column, and the grouping step nulls it. After review, the author plans to add two parser rules in parse_agg.c that limit which expressions become group Vars.

**User-visible surface.**
- Query results change for grouping sets with variable-free or constant-folded grouping expressions.
- New PlannerInfo field, to be moved to the end of the struct in back branches.

**What the author claims.**
- Fixes the 19 regression and the older bug without limiting constant folding or changing the aggregate executor.
- Distinct grouping columns keep separate IDs even when they fold to the same constant.
- Core regression, isolation and relevant TAP tests pass on master and REL_19_STABLE with assertions.
- Should go into REL_19_STABLE and back-patch to 18, which needs its own expected output.

**Thread state.** Shihao Zhong reviewed v1 and later showed that v3 breaks correct results (`SELECT 1 AS one, 1 AS uno GROUP BY ROLLUP(one)`, a HAVING case, a CUBE case). The author agrees v3 is unsafe, proposes two parser matching rules tied to bug #19752, and waits for Tom Lane's opinion before a new revision.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Planned matching rules redefine the bug the patch reports

fixable · level 1a, rubric item 1 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-2`

The v1/v3 commit message says `select (1 + 1) * 10 ... group by rollup (1 + 1)` should return NULL in the grand-total row. The second planned rule excludes grouping items without a level-zero Var from structural matching, so `1 + 1` inside `(1 + 1) * 10` is no longer a grouping reference and still returns 20. Users then cannot tell which result the project considers correct.

Message #8 accepts f,f for `now() is null ... rollup(now())` because the independently written expression is not a reference, but the commit message lists now() and constant expressions as cases that return wrong results. These two positions disagree on a user-visible contract, namely which expressions in the target list and HAVING count as grouping references. Rules that change query results in minor releases need one stated definition, checked against the standard and against 17 and 18 behaviour.

Thread: [#1 (Haibo Yan, 2026-10-10)](https://www.postgresql.org/message-id/CABXr29FSih8YtLCEe6Swquu_c%3DYvFiLVoH-ejvk2jVge9EMEMQ%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-20)](https://www.postgresql.org/message-id/CABXr29F4UDTaQVz7QZmVvLzz4R84ECP1%2Be6sVaSEWfbM6xKt%3Dw%40mail.gmail.com), [#7 (shihao zhong, 2026-10-21)](https://www.postgresql.org/message-id/CAGRkXqTbwvQKNaGuWwOvH0-DMTvdrC_wBRpu9GzQC5dOzgDMcA%40mail.gmail.com), [#8 (Haibo Yan, 2026-10-23)](https://www.postgresql.org/message-id/CABXr29EVOzjAseaNKO5xK2JrhQVP2UA89QfP3xw2A_gDOiNATA%40mail.gmail.com)

**Question:** Under the two planned parser rules, what does `select (1 + 1) * 10 from generate_series(1, 10) group by rollup (1 + 1)` return, and which documented rule says that is correct?

Counter-check: held. The commit message lists `(1 + 1) * 10 ... ROLLUP (1 + 1)` and now() as wrong results. The second rule in message #6 stops matching such subexpressions, and message #8 accepts f,f for now(), so the two definitions disagree.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Views in pg_rewrite keep group Vars made by the old parser matching, and the new planner code acts on them. |
| ABI / extension API | **yes** | The patch adds a PlannerInfo field, to be moved to the end of the struct in back branches. |
| SQL-standard conflict or non-standard syntax | **yes** | The planned parser rules change which target-list and HAVING expressions count as references to a grouping item. |
| new GUC | no |  |
| planner behaviour change | **yes** | mark_nullable_by_grouping() shares one PlaceHolderVar ID per RTE_GROUP column, which changes grouping-set query results. |
| pg_dump / restore round-trip | **yes** | A restore re-parses views under the new matching rules, so a view can return different rows than before the dump. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a back-patch through 18, and the fix now needs parser rules as well. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The planner change in var.c is correct only while parse_agg.c matches genuine grouping references. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Under the two planned parser rules, what does `select (1 + 1) * 10 from generate_series(1, 10) group by rollup (1 + 1)` return, and which stated rule makes that correct?
2. How does the back-patched planner avoid nulling group Vars that the old parser stored in existing views and rules on 18 and 19?
3. Which queries that return correct results on 18 today change results under the two parser rules, and is that acceptable in a minor release?
4. Can the two parser rules be back-patched to 18 and 19 in the same form as for master?
5. Will this fix be split from the patch for bug #19752, or wait for it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows grouping-set planning (RTE_GROUP, PlaceHolderVars, setrefs) and the grouping matching in parse_agg.c. The author waits for such an opinion, and none is in the visible thread.
- **Liability:** The owner signs for changed query results in minor releases of 18 and 19, for a PlannerInfo layout change in back branches, and for a lasting dependency between var.c and parser matching.
- **Author follow-up:** The author posted three versions in a few days, agreed that v3 is unsafe, and plans a new revision with parser rules.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The 19 regression gives wrong rows for grouping sets over NOT NULL columns, so the fix deserves attention now. The posted patch is small, but v3 is known to be unsafe and the parser rules that complete it are not posted.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-1` (fixable, level 2) Stored view parse trees keep the old group Var matching after back-patch. Not listed: level 2 is above the failing level 1a: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Back-patching parser matching changes to 18 changes released behaviour. Not listed: level 2 is above the failing level 1a: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Back-patched parser matching rules add a layer to grouping-clause code. Refuted (not specific): Duplicate of maintenance-fit-2, which asks the same question without precedents. [P6] and [P2] concern GROUP BY ALL, a new feature in beta, so they do not support an objection to a wrong-result fix.
- `standards-compat-1` Planner change also applies to stored views parsed under the old rules. Refuted (not specific): Duplicate of replication-ondisk-1, which states the stored-view case more directly. The catversion remark adds nothing, because a catversion bump does not re-parse stored rules.
- `maintenance-fit-1` Planner PHV sharing depends on a parser invariant nothing enforces. Refuted (no failure scenario): The scenario needs an unnamed future parser patch. The planner already nulls every group Var that contains a Var, so it relies on parser matching today, and message #8 plans regression tests for the independent-constant cases. [P6] concerns GROUP BY ALL.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes the same fix. The one objection rests on the planned parser rules, which no posted patch contains yet.
- standards-compat: 2 objection(s). Two objections raised. The remaining compatibility effects depend on parser rules that no posted patch contains yet.
- replication-ondisk: 1 objection(s). The new PlannerInfo field is a back-branch ABI point, and the commit message already plans to move it to the end of the struct. Nothing else touches WAL, on-disk format or the wire protocol.
- security: 0 objection(s). The proposal is a planner wrong-result fix for grouping sets. It adds no privilege, role, function, setting or exposure path, so a user sees nothing they could not see before. Wrong values in grand-total rows could in theory affect RLS or view predicates, but no such case is shown, and the fix only restores the correct NULL values.
- maintenance-fit: 2 objection(s). The bug is real and a fix is wanted. The objections concern how the combined planner and parser fix will be maintained and back-patched, not whether to fix it.
- already-exists: 0 objection(s). This is a wrong-result bugfix in planner internals. No existing feature, option or extension gives correct grouping-set nulling for variable-free grouping expressions, and the fix reuses the PHV-sharing approach of pullup_replace_vars_callback() and does not add a second mechanism.
- demand: 0 objection(s). This is a wrong-result bug fix, so the demand angle does not apply. In 19, common queries on NOT NULL columns return different rows than in 18, and two people in the thread show reproducible cases. Users who hit the bug cannot work around it without rewriting queries. Any remaining concerns are about the design (v3 is unsafe without the parser rules) and are for other angles.

### Precedents retrieved

- P1: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P2: [Revert "Add GROUP BY ALL".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f) (revert, Reverted, 2026-07-17) - used in the review
- P3: [Skip a redundant singleton GROUP BY node](https://commitfest.postgresql.org/patch/7419/) (cf, Withdrawn, 2026-10-09) - judged not relevant; reason summarised from its thread
- P4: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P5: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P6: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - used in the review; reason summarised from its thread
- P7: [remove_useless_groupby_columns is too enthusiastic](https://commitfest.postgresql.org/patch/3766/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P8: [Rename constants in preparation of variableness](https://commitfest.postgresql.org/patch/4828/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P9: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant; reason summarised from its thread
- P10: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P11: [Indeterminate behavior for the GROUP BY clause](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P12: [rewriteGraphTable: Fix missing RTEs in FROM clause by setting inFromCl=true](https://commitfest.postgresql.org/patch/6602/) (cf, Withdrawn, 2026-09-10) - judged not relevant
- P13: [Fix uninitialized variable access (freepages)](https://commitfest.postgresql.org/patch/3236/) (cf, Rejected, 2022-03-14) - judged not relevant
- P14: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P15: [Fix bug of COPY (on_error set_null)](https://commitfest.postgresql.org/patch/6774/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P16: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P17: [Optimizing NULL check on pkey column](https://commitfest.postgresql.org/patch/2699/) (cf, Returned with feedback, 2021-09-27) - judged not relevant
- P18: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - judged not relevant
- P19: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P20: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P21: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant
- P22: [Allowing control of upper/lower case folding of unquoted identifiers](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Exotic_Features) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant
- P24: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 138 s; cost reported by the CLI: $1.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 76 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:56:14.790Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
