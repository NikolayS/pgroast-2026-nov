# Design pre-review: extended statistic (examine_variable) ignored CollateExpr

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6397](https://commitfest.postgresql.org/patch/6397/) (Needs review); authors: Jian He (stayreal)
- Thread: [examine_variable ignored CollateExpr](https://www.postgresql.org/message-id/flat/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), started 2026-01-13
- Messages read: 10 of 10
- Patch prose read: v1-0001-proper-handling-examine_variable-RelabelType.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is sound: the planner should use expression statistics that match a COLLATE expression. The design is not settled. standards-compat-2 shows the v6 retry may match statistics built under a different collation, and maintenance-fit-3 shows the thread states no single matching rule. Planning cost is unmeasured for v6 after the cost objection in message #3 (maintenance-fit-1), and the stable target is not argued (standards-compat-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm whether the v6 retry compares collations, how much the second pass and the walker cost, or whether new helper signatures affect a back-patch.

## The proposal, restated

**Problem.** examine_variable strips every top-level RelabelType from the expression, including the one a COLLATE clause becomes. So a query that groups by `a COLLATE case_insensitive` uses the plain column statistics and not the extended statistics on that expression. In the author's demo the estimate is 52 rows and the actual count is 32. This affects users of nondeterministic collations and of expression statistics.

**Proposed design.** v1 changes the RelabelType handling in examine_variable in selfuncs.c. From v2 on, the reviewer Zizhuan Liu rewrote the patch. v6 first looks for an exact equal() match against index and statistics expressions, and only then retries with the top RelabelType stripped. It also merges adjacent RelabelTypes when strip_all_phvs_mutator removes PlaceHolderVars.

**User-visible surface.**
- Changed row estimates for queries with COLLATE on columns or expressions that have extended statistics or expression indexes.

**What the author claims.**
- With v1, the demo estimate changes from 52 rows to 32 rows, which matches the actual count.
- v3 replaces a second eval_const_expressions call with an in-place walker that should cost much less.
- v5 keeps the pre-7e9f852a lookup behaviour as a fallback, for compatibility.
- A reviewer measured no change in planning time for varchar columns with v5.

**Thread state.** Tom Lane called the v2 approach of a second eval_const_expressions call too expensive. Shihao Zhong found two estimate regressions, and v6 claims to fix both and restore the tests, but nobody has reviewed v6 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-01-13; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Fallback match may reuse statistics built under another collation

fixable · level 2, rubric item 12 · planner behaviour change · `standards-compat-2`

A user creates statistics on (a COLLATE case_insensitive) and runs GROUP BY a COLLATE "C". The exact match fails, and the retry strips the top RelabelType on both sides, so the two expressions match. The planner then uses the case-insensitive ndistinct of 32 for a grouping that returns 52 groups, which is the bug from message #1 in reverse.

Message #1 argues that ndistinct under one collation does not hold under another. Messages #7 and #9 keep the old strip-and-compare as a fallback for compatibility, and from v6 it also runs when only the statistics expression has a COLLATE. Nothing in the thread says whether the retry checks that the stripped collations agree, or limits itself to deterministic collations, where the statistics would be the same.

Thread: [#1 (jian he, 2026-01-13)](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#7 (ZizhuanLiu X-MAN, 2026-10-10)](https://www.postgresql.org/message-id/tencent_974EEA9CC644F88FB6235A8E89F3EF064B07%40qq.com), [#8 (shihao zhong, 2026-10-11)](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9 (ZizhuanLiu X-MAN, 2026-10-12)](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com)

**Question:** When the retry strips a COLLATE from either side, does it check that the collations used by the query and the statistics agree, or are both deterministic?

Counter-check: held. Message #7 says the retry strips the top RelabelType from both sides, and no message mentions a collation check. The stripping for non-Var expressions predates this patch, but whether v6 extends it to plain columns cannot be checked without the code.

### 2. Planning cost of v6 two-pass lookup is not measured

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `maintenance-fit-1`

A user plans many queries on varchar columns, whose Vars carry an implicit top-level RelabelType, against tables with several expression indexes and statistics objects. v6 compares every index and statistics expression twice when no exact match exists, and also runs a new walker inside strip_all_phvs_mutator. Planning time rises for users who never wrote COLLATE.

examine_variable runs for nearly every clause and grouping expression the planner estimates. In message #3, Tom Lane objected to the cost of v2. The only measurement, in message #8, covers v5 on varchar columns with no index or statistics expressions present. v6 changed the retry conditions and was not measured.

Thread: [#3 (Tom Lane, 2026-08-16)](https://www.postgresql.org/message-id/453227.1786895971%40sss.pgh.pa.us), [#8 (shihao zhong, 2026-10-11)](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9 (ZizhuanLiu X-MAN, 2026-10-12)](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com)

**Question:** What does planning time look like with v6 for varchar and text columns on tables with many expression indexes and statistics objects, with and without PlaceHolderVars?

Counter-check: held. The only measurement is Shihao Zhong's in message #8, for v5 on varchar columns. v6 changed the retry conditions, and nobody measured it or the new walker after Tom Lane's cost objection in message #3.

### 3. Back-patching a planner estimate change to stable branches

fixable · level 2, rubric item 6 · backpatchability, planner behaviour change · `standards-compat-1`

A site with expression statistics or expression indexes on COLLATE expressions installs a minor release. Row estimates for those queries change, for example 52 to 32 or 100 to 26, and some plans flip. The site finds out from slower queries after the update, with no setting to turn the change off.

The CommitFest entry targets stable, but every version since message #6 changes which statistics a COLLATE query matches. Messages #6 and #8 show that each version so far moved estimates for tables the patch did not aim at. Minor releases normally avoid changes to plans unless a wrong result or a crash forces one. The thread does not say which branches have the bug or why a back-patch is worth the plan churn.

Thread: [#1 (jian he, 2026-01-13)](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#6 (shihao zhong, 2026-10-05)](https://www.postgresql.org/message-id/CAGRkXqTZjRj2x0vKFbT6pM0%2B-_1kvKbmSPA9HwCX_wB-BkmynA%40mail.gmail.com), [#8 (shihao zhong, 2026-10-11)](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9 (ZizhuanLiu X-MAN, 2026-10-12)](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com)

**Question:** Which stable branches give the wrong estimate, and why should a plan-changing estimate fix go there instead of master only?

Counter-check: held. The CommitFest entry targets stable, and the thread does not name the affected branches or argue for a back-patch. Messages #6 and #8 show estimates moving for cases the patch did not aim at, so the plan-change risk is concrete.

### 4. Ad hoc matching rules in examine_variable without an engaged owner

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-3`

A later patch changes how expressions in index or statistics definitions are normalized, for example how CollateExpr is stripped. That patch has to keep the exact pass, the retry pass and the per-side RelabelType flags consistent. If it misses one, estimates change without any test failing, and users report it releases later.

v6 adds three flags that decide which comparisons the retry runs. The retry keeps the old behaviour from before 7e9f852a as a compatibility fallback, and the thread does not state the rule it implements. Each of v4, v5 and v6 fixed a case the previous version broke, which shows the rules are easy to get wrong. The original author has been silent since message #1, Richard Guo was CC'd in message #4 and did not reply, and no committer has reviewed the rewrite.

Thread: [#1 (jian he, 2026-01-13)](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#4 (ZizhuanLiu X-MAN, 2026-08-17)](https://www.postgresql.org/message-id/tencent_212DD484ECF7882505B551630CFC74DB0C06%40qq.com), [#7 (ZizhuanLiu X-MAN, 2026-10-10)](https://www.postgresql.org/message-id/tencent_974EEA9CC644F88FB6235A8E89F3EF064B07%40qq.com), [#9 (ZizhuanLiu X-MAN, 2026-10-12)](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com)

**Question:** Can the rule for when an expression matches statistics be stated once, so that index and statistics normalization and examine_variable share one helper and do not diverge?

Counter-check: held. Message #9 describes three per-side flags, and the thread states no single matching rule. The silence of the original author and Richard Guo is a committability point and should not add weight to the design objection.

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
| planner behaviour change | **yes** | The patch changes which statistics examine_variable matches for COLLATE expressions, so row estimates and plans change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and the change alters row estimates in minor releases. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v6 adds an exact pass, a retry pass and per-side RelabelType flags that later normalization changes must keep consistent. |
| performance on the hot path | **yes** | examine_variable runs for most estimated clauses, and v6 adds a second comparison pass and a walker in strip_all_phvs_mutator. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. When the v6 retry strips a top RelabelType from either side, does it check that the query collation and the statistics collation agree, or that both are deterministic?
2. Does v6 still change the message #1 demo estimate from 52 to 32 rows?
3. What is the planning time with v6 for varchar and text columns on tables with many expression indexes and statistics objects, with and without PlaceHolderVars?
4. Which stable branches give the wrong estimate, and why should a plan-changing estimate fix go there instead of master only?
5. Can the rule for when a query expression matches an index or statistics expression be stated once and implemented in one shared helper?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows selfuncs.c and extended statistics matching. Tom Lane objected once to the cost of v2 in message #3, and no committer has commented on v3 to v6.
- **Liability:** The committer signs for every estimate change on COLLATE queries, in back branches too if the stable target stays, and for keeping the two-pass matching rules consistent with index and statistics normalization.
- **Author follow-up:** The original author has not replied since message #1. The reviewer Zizhuan Liu wrote v2 to v6 and answered each reported regression, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The fix helps users who combine COLLATE expressions with expression statistics or indexes, and it changes a planner path that every query uses. The patch is small, but v4, v5 and v6 each fixed a regression from the previous version, and no code was read for this review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Back-patching estimate changes risks plan flips in minor releases. Refuted (not specific): Duplicate of standards-compat-1, which states the same back-patch concern with more specific evidence.
- `demand-1` Back-patching estimate changes needs field demand the thread does not show. Refuted (already handled): Message #8 confirms that v5 restores rows=100 for the plain-column COLLATE case without statistics, so the stated scenario no longer occurs. The remaining back-patch concern duplicates standards-compat-1, and a reproducible misestimate that ignores user-created statistics needs no field report.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier rejected, returned, withdrawn or reverted attempt at this idea. Message #7 cites commit 7e9f852a as the change that added the stripping v6 revisits, but the thread does not report it as rejected or reverted.
- standards-compat: 2 objection(s). Both objections are fixable. The proposal adds no syntax, GUC, catalog object or dump format, so pg_dump round-trip, psql conventions and the SQL standard are not affected.
- replication-ondisk: 0 objection(s). The patch changes only planner estimation logic in selfuncs.c and regression tests. It adds no on-disk format, WAL record, catalog column, wire protocol message or persistent state, so this angle has nothing to object to. Back-patch ABI risk depends on whether new helper signatures are exported, which cannot be checked without code.
- security: 0 objection(s). This proposal only changes how examine_variable in selfuncs.c matches a query expression against index and extended statistics expressions. It does not change who can create statistics, read them, or run anything, so it adds no privilege model, no new object and no new code path that runs user-controlled code as another role. The existing checks on whether a user may use statistics through operators that are not leakproof sit outside the matching step, and the proposal does not say the patch touches them. Without the code I could not check that the new retry path still goes through those checks.
- maintenance-fit: 3 objection(s). The goal of using matching expression statistics is reasonable. The objections are about cost, back-patching and how hard the matching rules are to maintain.
- already-exists: 0 objection(s). This is a bug fix to how examine_variable matches expression statistics. No existing feature, setting or extension lets the planner use expression statistics defined with a COLLATE clause. Users can only avoid writing COLLATE, which changes query semantics, so this angle has nothing to raise.
- demand: 1 objection(s). The fix repairs a lookup that misses existing expression statistics, so it is desirable for master. The objection is limited to the stable-branch target, where no field demand is shown.

### Precedents retrieved

- P1: [Extended statistics / estimate Var op Var clauses](https://commitfest.postgresql.org/patch/2831/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [Support regular expressions with nondeterministic collations](https://commitfest.postgresql.org/patch/5334/) (cf, Withdrawn, 2024-12-30) - judged not relevant; reason summarised from its thread
- P3: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Self contradictory examining on rel's baserestrictinfo](https://commitfest.postgresql.org/patch/5404/) (cf, Returned with feedback, 2025-01-23) - judged not relevant; reason summarised from its thread
- P5: [Make selectivity example match wording](https://commitfest.postgresql.org/patch/3695/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P6: [Revert "pg_collation_actual_version() -> pg_collation_current_version()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8556267b2b1b8e1c26037c4c25cf390ee5afb5d9) (revert, Reverted, 2021-02-26) - judged not relevant
- P7: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P8: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant; reason summarised from its thread
- P9: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P10: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant; reason summarised from its thread
- P11: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [pg_hba_lookup function to get all matching pg_hba.conf entries](https://commitfest.postgresql.org/patch/356/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P13: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P14: [use regular expressions to simplify less_greater and not_equals](https://commitfest.postgresql.org/patch/3281/) (cf, Rejected, 2021-09-05) - judged not relevant
- P15: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P16: [Typos in the comment for the estimate_multivariate_ndistinct()](https://commitfest.postgresql.org/patch/5740/) (cf, Withdrawn, 2025-05-07) - judged not relevant
- P17: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P18: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P19: [Better estimate merging for duplicate vars in clausesel.c](https://commitfest.postgresql.org/patch/1036/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P20: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P21: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P22: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P23: [Collation code refactoring](https://commitfest.postgresql.org/patch/4058/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P24: [Adjust tuples estimate for appendrel](https://commitfest.postgresql.org/patch/4711/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 128 s; cost reported by the CLI: $1.90
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 73 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:50:20.734Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
