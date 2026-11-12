# Design pre-review: BUG #19798: "failed to build any 4-way joins" with ROLLUP over a LATERAL subquery that references the nullable s

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7601](https://commitfest.postgresql.org/patch/7601/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [BUG #19798: "failed to build any 4-way joins" with ROLLUP over a LATERAL subquery that references the nullable s](https://www.postgresql.org/message-id/flat/19798-5a5da856f5f71118%40postgresql.org), started 2026-10-26
- Messages read: 2 of 2
- Patch prose read: 0001-Fix-failed-to-build-any-4-way-joins.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real, reproducible and well analysed, and a fix is wanted. The one surviving objection, maintenance-fit-1, is a fixable design point: the patch adds a second rule in join_search_one_level() next to has_join_restriction(), and message #2 does not say whether the three join.sql plan changes of the alternative are worse. A committer who knows the join search code has to choose between the two fixes.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the exact condition of the exception, its cost when the PlaceHolderVar list is long, and whether it admits extra pairs that change plans on stable branches could not be judged. The prose does not show whether a regression test is included.

## The proposal, restated

**Problem.** A query with grouping sets fails at plan time with "failed to build any 4-way joins" when a LATERAL subquery over two or more tables returns a column from the nullable side of an earlier outer join. The reporter bisected it to commit 90947674fc9 (2018) and reproduced it on 14 through master. The same query without grouping sets plans fine.

**Proposed design.** With grouping sets, the pulled-up lateral output becomes a PlaceHolderVar evaluated at {c,d} that needs values from the completed b/a outer join, so the only legal final join is the bushy pair {b,a} + {c,d}. The bushy-join loop in join_search_one_level() skips {b,a} as old_rel because it has no join clauses and has_join_restriction() is false. The patch adds a local exception there for a joinrel that supplies all of a PHV's ph_lateral rels while the PHV's ph_eval_at rels are not yet joined, and leaves the pair decision to have_join_order_restriction().

**What the author claims.**
- The fix is limited to bushy-join pruning in joinrels.c and keeps the existing pruning heuristic.
- Standard regression tests pass with no plan changes.
- Removing the preliminary check also fixes the bug but makes the planner examine more bushy-join pairs.
- Changing has_join_restriction() also fixes the bug but changes three plans in join.sql and affects other parts of join search.

**Thread state.** Only the reporter and the patch author have posted, and nobody has reviewed the patch yet. The author asks for thoughts on the chosen approach.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Local pruning exception duplicates the meaning of has_join_restriction()

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-1`

A later patch changes how PlaceHolderVars or lateral references restrict join order and updates has_join_restriction() only. The bushy-loop exception keeps the old rule. A new "failed to build any N-way joins" error then appears on some grouping-sets or lateral query, found later by a fuzzer or a user.

After this patch, the join search has two places that decide whether a rel takes part in a join-order restriction: has_join_restriction() and the new exception in join_search_one_level(). Nothing fails if they drift apart. Message #2 rejects fixing has_join_restriction() because three join.sql plans change, but it does not say whether those plans get worse. A committer would likely prefer one definition of "has a join restriction", or a fix to the joinrel lateral_referencers that message #1 says is always NULL.

Thread: [#1 (PG Bug reporting form, 2026-10-26)](https://www.postgresql.org/message-id/19798-5a5da856f5f71118%40postgresql.org), [#2 (Tender Wang, 2026-10-27)](https://www.postgresql.org/message-id/CAHewXNkpyvue6dx5Q33mHXnTh4uxDHHAw0szbwGCBnyZwHThpA%40mail.gmail.com)

**Question:** Are the three join.sql plan changes from the has_join_restriction() fix actually worse, and if not, why keep a second, local rule?

Counter-check: held. Message #2 explains the choice by three join.sql plan changes and wider effects, but does not say whether those plans are worse. The patch adds a second rule that must track has_join_restriction() and have_join_order_restriction(), and nothing fails if they drift.

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
| planner behaviour change | **yes** | The patch changes which bushy-join pairs join_search_one_level() considers in joinrels.c. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug reproduces on 14 through master, and the thread does not name the target branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The new exception is a second join-restriction rule that must stay in step with has_join_restriction(). |
| performance on the hot path | **yes** | The new check runs in the bushy-join loop of join search, and its planning-time cost is not measured. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Are the three join.sql plan changes from the has_join_restriction() fix worse, equal or better than the current plans?
2. Why is a local exception in join_search_one_level() preferable to setting lateral_referencers for joinrels, which message #1 says is always NULL?
3. Which branches should receive the fix, given that the bug reproduces on 14 through master?
4. Does the patch fix the 5-way and 6-way variants from message #1, and does it add regression tests for them?
5. What is the planning-time cost of the new check on queries with many relations and many PlaceHolderVars?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the join search code in joinrels.c and how PlaceHolderVars and lateral references restrict join order. No such committer has posted in the two visible messages.
- **Liability:** The owner signs for join-order behaviour on all supported branches if back-patched, including any plan change or new "failed to build any N-way joins" case, and for keeping the new exception in step with has_join_restriction().
- **Author follow-up:** The author reproduced the bug, compared three fixes and asked for thoughts in message #2, which suggests he will follow up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 2.1 KB in one file and fixes an internal planner error on valid SQL that reaches back to 14. The failing query shape is rare and was found by a fuzzer, but the choice between the three fixes needs a reviewer who knows join search.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no row about this idea. Message #1 points to an earlier "failed to build any N-way joins" fix and says this query still fails with it applied, so that fix does not cover this case. No earlier rejection reason applies here.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch is a planner bug fix in joinrels.c with no new syntax, GUC, catalog object, psql change or dump format. Queries that now fail with an internal error would plan, and the author reports no plan changes in the regression tests. Back-patch scope and the plan stability of existing queries on stable branches could not be checked without the code.
- replication-ondisk: 0 objection(s). The patch changes only join search pruning in joinrels.c. It touches no on-disk format, WAL record, catalog, wire protocol or exported struct, so it holds no persistent or cross-node state. One ABI point is open: a back-patch to 14 must not change exported planner function signatures or RelOptInfo layout, and the prose does not say whether it does.
- security: 0 objection(s). The patch changes only bushy-join pruning in the planner's join search. It adds no privilege, role, function, view or setting, and exposes no new data. An unprivileged user can already send queries with many relations, so the extra pairs the patch admits are not a new denial-of-service path. The security angle finds no objection.
- maintenance-fit: 1 objection(s). This is a narrow planner bug fix in one file. It adds no user-visible surface. If the PHV list is empty, the extra check should cost almost nothing on the common path. Only the duplicated-rule concern applies from this angle.
- already-exists: 0 objection(s). This is a planner bug fix for an internal error, so no existing feature, extension or idiom meets the need. Setting join_collapse_limit = 1 and from_collapse_limit = 1 avoids the error, but only by giving up join reordering, so it is a workaround. The patch extends the existing pruning check in join_search_one_level() and adds no second mechanism.
- demand: 0 objection(s). This is a bug fix for an internal planner error ("failed to build any 4-way joins") on valid SQL. The report includes a reproducer, a bisect to a 2018 commit, and failures on 14 through master. Demand for a fix to a plan-time ERROR on legal queries needs no further evidence. Design and cost questions belong to other angles.

### Precedents retrieved

- P1: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P2: [Add a greedy join search algorithm to handle large join problems](https://commitfest.postgresql.org/patch/6486/) (cf, Withdrawn, 2026-02-14) - judged not relevant; reason summarised from its thread
- P3: [Revert "Replace pg_restrict by standard restrict"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fa16e7fd84886643f480c36614fa11d45fc6d26f) (revert, Reverted, 2026-01-14) - judged not relevant
- P4: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P5: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - judged not relevant
- P6: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P7: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant; reason summarised from its thread
- P8: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant; reason summarised from its thread
- P9: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P10: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant; reason summarised from its thread
- P11: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P12: [Support run-time partition pruning for hash join](https://commitfest.postgresql.org/patch/4512/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P13: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant
- P14: [Revert ill-considered restriction of dtrace support to Solaris only.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=db13a81ab44f5ead85a4e77b30006c81bb6d20af) (revert, Reverted, 2009-09-04) - judged not relevant
- P15: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant
- P16: [Allow placeholders in ALTER ROLE w/o superuser](https://commitfest.postgresql.org/patch/3749/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P18: [Disallow outer-level and WHERE-clause aggregates in GRAPH_TABLE](https://commitfest.postgresql.org/patch/7144/) (cf, Withdrawn, 2026-09-08) - judged not relevant
- P19: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [Revert changes about warnings/errors for placeholders.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cab5b9ab2c066ba904f13de2681872dcda31e207) (revert, Reverted, 2021-12-27) - judged not relevant
- P21: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P22: [Detect the null type correctly in setop case by peeking the later exprs.](https://commitfest.postgresql.org/patch/2653/) (cf, Withdrawn, 2020-07-23) - judged not relevant
- P23: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P24: [Revert "Fix an O(N^2) problem in foreign key references".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3d9e8db9e56beb0501d168779ae9644587924061) (revert, Reverted, 2015-09-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 113 s; cost reported by the CLI: $1.32
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:02.705Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
