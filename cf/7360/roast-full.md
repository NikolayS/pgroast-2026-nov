# Design pre-review: Skip LEFT/ANTI joins to a provably empty inner rel

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7360](https://commitfest.postgresql.org/patch/7360/) (Needs review); authors: Ilia Evdokimov (ilidock95)
- Thread: [Skip LEFT/ANTI joins to a provably empty inner rel](https://www.postgresql.org/message-id/flat/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com), started 2026-09-29
- Messages read: 1 of 1
- Patch prose read: v1-0001-Skip-outer-joins-to-a-provably-empty-inner-relati.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No surviving objection is fatal, and none stops review at levels 1 or 2. Three fixable points remain: demand-1 asks for a realistic workload, standards-compat-1 asks whether the "stable" target means a back-patch, and maintenance-fit-3 asks for a comparison with a selectivity-only fix. A committer who knows outer-join Var handling has to judge whether the path substitution is safe. Confidence is low because only message #1 exists and no code was read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so this review could not check whether outer paths emit the join's reltarget correctly when the outer side is a join, an Append, or contains PlaceHolderVars. It also could not check test coverage for the all-partitions-pruned case.

## The proposal, restated

**Problem.** When the inner side of a LEFT or ANTI join is proven empty, the planner still builds a real join against the dummy rel. The empty side is rescanned once per outer row, and a pushed-down "inner.x IS NULL" qual gets default selectivity (0.005), so the join is underestimated by orders of magnitude and the joins above it are planned badly.

**Proposed design.** A new step in populate_joinrel_with_paths() handles JOIN_LEFT and JOIN_ANTI with a dummy inner rel when every pushed-down qual is "innervar IS NULL" and the reltarget can be computed from the outer rel alone. The join then takes the outer rel's size, and projections of all unparameterized and partial outer paths are added beside the regular join paths, so they must win on cost. FULL JOIN is not handled.

**What the author claims.**
- Row estimates for such joins become correct, which avoids bad plans for the joins above them.
- The example query drops from about 425 ms to about 150 ms, with a hash join replacing a nested loop over 1M rows.
- Keeping all outer paths preserves sort orders, so ORDER BY ... LIMIT over an index still works.
- Adding partial outer paths lets parallel plans avoid the per-row nested loop.
- RIGHT joins are covered because they are planned as LEFT joins with the sides swapped.

**Thread state.** Only the first message is visible. Nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No real-world query shown that hits an empty inner rel

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), planner behaviour change · `demand-1`

A committer adds a new path-generation step to populate_joinrel_with_paths() and owns its interaction with nulling relids, partial paths and setrefs. If few production queries ever reach a LEFT or ANTI join whose inner rel is dummy and whose pushed-down quals are all IS NULL, the project keeps that code and its corner cases for little gain.

The only demonstration in message #1 uses a literal "AND false" in the ON clause, which a user can remove by hand. The partition-pruning case is named in one sentence but has no example, plan or field report. No -general or -bugs report, no user request and no other database are cited, so the size of the user group is unknown. The FULL JOIN attempt in message #1 failed with "wrong varnullingrels", which shows this area has real correctness risk to set against the gain.

Thread: [#1 (Ilia Evdokimov, 2026-09-29)](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com)

**Question:** Which real workloads, such as ORM-generated queries or LEFT JOINs to partitioned tables pruned to nothing, produce this plan shape, and how often do users report the bad estimate?

Counter-check: held. The only worked example uses a literal "AND false", and the partition-pruning case has no plan or report behind it. The failed FULL JOIN attempt is not part of this proposal and should not count as support; the request for demand evidence stands without it.

### 2. CommitFest target "stable" implies a plan change in minor releases

fixable · level 1b, rubric item 6 · backpatchability, planner behaviour change · `standards-compat-1`

A user applies a minor release that carries this patch. Row estimates for LEFT and ANTI joins to empty rels change, so the plans for the joins above them change too. A query that was tuned or stable on the old plan picks a different join order or method, and the user finds out from a regression in production after the upgrade.

CommitFest entry 7360 sets the target version to "stable". The patch changes cardinality estimates and adds new path candidates in populate_joinrel_with_paths(), and the before/after plans in message #1 show a different plan shape. Back branches normally avoid planner changes that alter plans for existing queries without a bug that produces wrong results. Message #1 and the commit message frame the change as a performance improvement and do not ask for a back-patch, so the target may be wrong.

Thread: [#1 (Ilia Evdokimov, 2026-09-29)](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com)

**Question:** Is this intended for master only, and if a back-patch is wanted, what makes this estimate change a bug fix rather than a plan change for existing queries?

Counter-check: held. The CommitFest entry says "Target version: stable", and message #1 presents an optimization that changes estimates and plan shapes. The proposal does not say the patch is for master only, so the question stands as a fixable metadata point.

### 3. A narrower estimate fix may give most of the gain

fixable · level 2, rubric item 12 · design, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

The project adds a second path-level form of join removal, next to the existing left-join removal in analyzejoins.c. Later, a planner developer changes join removal or outer-join estimation and has two separate places to keep consistent. Users see different plans depending on which mechanism handled the join.

Message #1 names the row estimate as "the bigger issue". The cost of rescanning a one-time-false Result for each outer row is small. Making the selectivity of an "innervar IS NULL" qual equal to 1 when the inner rel is dummy would fix the estimate in the example without adding joinrel paths. The proposal does not compare against that option or say what the path substitution adds beyond it.

Thread: [#1 (Ilia Evdokimov, 2026-09-29)](https://www.postgresql.org/message-id/cba47b16-76cf-4d84-9dea-2421ccf83eee%40tantorlabs.com)

**Question:** How much of the gain in the example remains if only the IS NULL selectivity over a dummy inner rel is corrected and no outer paths are added?

Counter-check: held. Message #1 calls the estimate "the bigger issue", and its own plan shows the left join taking about 38 ms of 425 ms. The proposal does not compare the path substitution with a selectivity-only fix, so the design question stands.

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
| planner behaviour change | **yes** | The patch changes row estimates and adds outer-rel paths for LEFT and ANTI joins to a dummy inner rel. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry sets the target version to "stable" for a change that alters plans. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The new step in joinrels.c depends on nulling-relid rules that later outer-join work must keep in mind. |
| performance on the hot path | **yes** | The new check sits in populate_joinrel_with_paths(), and message #1 reports no planning-time numbers for queries with many joins. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is the patch meant for master only, and if a back-patch is wanted, what makes this estimate change a bug fix?
2. Which real workloads reach a LEFT or ANTI join with a dummy inner rel, for example a partitioned table with every partition pruned by the ON clause?
3. How much of the gain in the example remains if only the IS NULL selectivity over a dummy inner rel is corrected and no outer paths are added?
4. What is the planning-time cost of the new step for queries with many joins that do not qualify?
5. Were pushed-down quals that reference only outer Vars considered, and why does the patch not handle them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the planner's join path generation and the nulling-relid rules for outer-join Vars. Nobody has replied to message #1 yet.
- **Liability:** The owner signs for the correctness of using outer-rel paths as joinrel paths, including any later "wrong varnullingrels" failure. That liability lasts as long as the code stays.
- **Author follow-up:** No sign either way; the thread has only message #1.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is about 11 KiB and changes one planner file plus tests, but its safety rests on nulling-relid rules that few reviewers know well. The only worked example uses a literal "AND false", so the number of affected users is unknown.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Target version "stable" implies back-patching a planner behaviour change. Refuted (not specific): Duplicate of standards-compat-1, which makes the same point and also allows that the target field may be a mistake.
- `maintenance-fit-2` Correctness depends on subtle nulling-relid reasoning in a new place. Refuted (already handled): Message #1 states the invariant: pull_varnos() of the reltarget, including nulling relids, must be a subset of the outer relids, and it explains why FULL JOIN breaks that. The scenario needs an unnamed future patch, and the setrefs check would fail loudly.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] (LEFT JOIN removal via DISTINCT, stopped on volatility checks) and [P2] (inner join removal via foreign keys and an AlternativePlan node) share keywords only, and their recorded reasons do not apply to skipping a join whose inner rel is already dummy. Message #1 does not mention any earlier discussion of this idea.
- standards-compat: 1 objection(s). The patch adds no syntax, no GUC, no catalog object and nothing pg_dump must handle. Changed plans and EXPLAIN output on master are the intended effect and follow existing join-removal practice.
- replication-ondisk: 0 objection(s). The patch changes only planner path generation in joinrels.c and adds no persistent or cross-node state: no WAL, on-disk, catalog, protocol or exported struct changes. The "stable" target and plan changes in minor releases fall under other angles, since nothing here blocks a back-patch on ABI grounds.
- security: 0 objection(s). The proposal has no privilege or exposure aspect, so this angle has no objection. It is a planner-only change to paths and row estimates in joinrels.c, with no new SQL, GUC, function or catalog. Privilege checks on the dropped inner rel come from the range table, as with existing left join removal, and the only quals involved are "innervar IS NULL", which call no user functions, so RLS and leakproofness are not affected.
- maintenance-fit: 3 objection(s). Hot-path cost looks small, since the check runs only when the inner rel is already dummy. The objections above focus on back-patching, fragility, and the choice of design.
- already-exists: 0 objection(s). Postgres has no existing way to get this result. Left join removal in analyzejoins.c runs before relation sizes are set and dummy rels are known, and it does not apply here because the IS NULL qual uses an inner Var. set_join_pathlist_hook can add paths but cannot fix the joinrel row estimate, which is the main gain. Rewriting the query is not possible when pruning empties the inner rel at plan time.
- demand: 1 objection(s). The one objection asks for evidence of demand. Costs appear to fall only on queries with a dummy inner rel, so the main cost is maintenance, not planning time for everyone.

### Precedents retrieved

- P1: [Remove LEFT JOINs in more cases](https://commitfest.postgresql.org/patch/1364/) (cf, Returned with feedback, 2018-09-28) - used in the review; reason summarised from its thread
- P2: [INNER JOIN removals](https://commitfest.postgresql.org/patch/130/) (cf, Returned with feedback, 2015-05-11) - used in the review; reason summarised from its thread
- P3: [INNER JOIN removals](https://commitfest.postgresql.org/patch/27/) (cf, Rejected, 2015-02-14) - judged not relevant; reason summarised from its thread
- P4: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - judged not relevant
- P5: [Memoize ANTI and SEMI JOIN inner](https://commitfest.postgresql.org/patch/5636/) (cf, Withdrawn, 2025-06-30) - judged not relevant; reason summarised from its thread
- P6: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant; reason summarised from its thread
- P7: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P8: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P9: [Plan a filtering inner join as a semijoin](https://commitfest.postgresql.org/patch/7202/) (cf, Withdrawn, 2026-08-27) - judged not relevant; reason summarised from its thread
- P10: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P11: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/1247/) (cf, Withdrawn, 2021-03-11) - judged not relevant
- P12: [Disallow outer-level and WHERE-clause aggregates in GRAPH_TABLE](https://commitfest.postgresql.org/patch/7144/) (cf, Withdrawn, 2026-09-08) - judged not relevant
- P13: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P14: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P16: [Support run-time partition pruning for hash join](https://commitfest.postgresql.org/patch/4512/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P17: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant
- P18: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5) (revert, Reverted, 2006-04-09) - judged not relevant
- P20: [Push aggregation down to base relations and joins](https://commitfest.postgresql.org/patch/3764/) (cf, Returned with feedback, 2024-01-22) - judged not relevant
- P21: [PoC: improving join estimates using sampling](https://commitfest.postgresql.org/patch/3211/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P22: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2954/) (cf, Withdrawn, 2021-02-19) - judged not relevant
- P23: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P24: [Select Reference Page - Make Join Syntax More Prominent](https://commitfest.postgresql.org/patch/3726/) (cf, Returned with feedback, 2022-09-10) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 137 s; cost reported by the CLI: $1.10
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 68 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:44:34.129Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
