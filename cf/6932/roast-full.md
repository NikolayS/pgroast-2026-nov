# Design pre-review: Improve row estimation with multi-column unique indexes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6932](https://commitfest.postgresql.org/patch/6932/) (Needs review); authors: Anthonin Bonnefoy (bonnefoa)
- Thread: [Improve row estimation with multi-column unique indexes](https://www.postgresql.org/message-id/flat/CAO6_XqoiwRoVFS1x1oaUpwUi5kKwOj1S1JVYBO2QX91ORJJoxg%40mail.gmail.com), started 2026-06-24
- Messages read: 3 of 3
- Patch prose read: v3-0002-Adjust-row-estimation-and-index-selectivity-with-.patch, v3-0003-Avoid-building-indexBoundQuals-if-path-is-unique.patch, v3-0001-Added-planner-estimation-test-for-multi-column-ro.patch, v1-0001-Adjust-row-estimation-and-index-selectivity-with-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is accepted by the only reviewer, and no objection survived at level 1. Two level 2 objections survived. standards-compat-1: v3 gives an index path rows=1 while the relation estimate stays high, and message #3 does not answer the README rule raised in message #2. standards-compat-3: v2 and v3 disagree on deferrable unique constraints. The thread needs to settle both before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Not judged: whether is_path_unique handles IS NULL, arrays and partial-index predicates correctly, and which callers read the path row count that v3 changes.

## The proposal, restated

**Problem.** When equality clauses cover all key columns of a multi-column unique index, the planner multiplies per-column selectivities and can estimate many rows (2149 in the example). The inflated cost can make the planner pick another index, such as one matching ORDER BY, over the unique lookup. This affects any query that looks up rows by a composite unique key.

**Proposed design.** v3 adds a "unique path" flag on IndexPath, set by a new is_path_unique when index paths are built: a unique index with at least one equality clause on every key column. cost_index then sets the path rows to 1, genericcostestimate sets selectivity to 1.0/reltuples, and btcostestimate uses the flag for numIndexTuples = 1. v2 (Chengpeng Yan) instead clamps the relation-level estimate in set_baserel_size_estimates() and get_parameterized_baserel_size().

**User-visible surface.**
- Changed planner row estimates and plan choice for equality lookups on all key columns of a multi-column unique index.
- C API: new unique_path field in IndexPath, visible to extensions that build or inspect index paths.
- Equality lookups on the full key of a multi-column unique index are treated as returning at most one row, with no new SQL syntax, GUC or catalog column.

**What the author claims.**
- In the example the estimate drops from 2149 rows to 1, and the planner keeps the unique index instead of switching to an index on id.
- A relation-level clamp, as in v2, does not work for unique partial indexes, so the one-row bound belongs to the path.
- IS NULL counts as equality when the index uses NULLS NOT DISTINCT.
- For deferrable unique constraints, a one-row estimate is usually closer to the truth than a statistics-based estimate.
- When the path is unique, most selectivity work can be skipped (0003).

**Thread state.** Chengpeng Yan agreed the problem is real but said v1 lowered rows at the wrong level and posted v2 with a relation-level clamp. The author answered with v3, a path-level design, and nobody has replied to v3 since July.

## 2. Prior attempts

### Revert "Use Foreign Key relationships to infer multi-column join selectivity". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb)
- **Date:** decided 2016-06-07; started 2016-04-07
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit reverts 137805f89 as well as the associated commits 015e88942, 5306df283, and 68d704edb. We found multiple bugs in this feature, and there was concern about possible planner slowdown (though to be fair, exhibiting a very large slowdown proved difficult). The way forward requires a considerable rewrite, which may or may not be possible to accomplish in time for beta2. In my judgment reviewing the rewrite will be easier to accomplish starting from a clean slate, so let's temporarily revert what's there now. This also leaves us in a safe state if it turns out to be necessary to postpone the rewrite to the next development cycle.
- **Reason (quote):** "We found multiple bugs in this feature, and there was concern about possible planner slowdown (though to be fair, exhibiting a very large slowdown proved difficult)."
- **What is different this time:** P24 used foreign keys to infer join selectivity across relations. This proposal uses unique indexes to bound one relation's scan to one row, with a much smaller scope.


## 3. Objections a reviewer is likely to raise

### 1. Paths of one relation get different row counts for the same clauses

fixable · level 2, rubric item 12 · planner behaviour change · `standards-compat-1`

A query joins visitor on (org, name). The index scan path reports rows=1, while the seq scan, bitmap path and the join size, which use the relation estimate, still report about 2149. EXPLAIN shows a scan with rows=1 under a join sized from 2149, and the user cannot tell which estimate the planner acted on.

Message #2 says the optimizer README expects paths for the same relation and parameterization to share one row count. v3 keeps the path-level design and sets rows only on the IndexPath. Message #3 answers with partial unique indexes but says nothing about the README rule or about join estimates. A committer would likely want a relation-level clamp where it is valid, plus a path-level clamp only for partial indexes, with a stated reason for that exception.

Thread: [#2 (Chengpeng Yan, 2026-06-28)](https://www.postgresql.org/message-id/0AF0607F-5FC9-46BD-B593-DE78BEBAAA36%40outlook.com), [#3 (Anthonin Bonnefoy, 2026-07-01)](https://www.postgresql.org/message-id/CAO6_Xqpg9t6n0iibUwqihfEx3xPwWtRFGenxi%3DtKWoCNqyCcGA%40mail.gmail.com)

**Question:** For which index types and plan shapes may index paths report fewer rows than the relation, and how do join size estimates stay consistent with that?

Counter-check: held. The optimizer README requires equal row counts for paths of one relation and parameterization, and join size estimates use the relation-level row count. Message #3 answers only the partial-index point, so the inconsistency is unanswered.

### 2. Deferrable unique constraints changed without a decision in the thread

fixable · level 2, rubric item 7 · planner behaviour change · `standards-compat-3`

A transaction loads rows under a deferred unique constraint and then joins against the table on the full key before commit. The planner assumes one row per key and picks a nested loop. Duplicates exist until commit, so the join runs much slower than planned, and nothing in EXPLAIN explains why.

v2 excludes deferrable constraints. Message #3 keeps them and argues that one row is usually closer to the truth. That argument is reasonable, but it extends the B-tree shortcut to multi-column keys without a recorded decision. A committer would likely want the choice made explicitly and documented, because this is the case where the estimate can be wrong by the size of the batch.

Thread: [#2 (Chengpeng Yan, 2026-06-28)](https://www.postgresql.org/message-id/0AF0607F-5FC9-46BD-B593-DE78BEBAAA36%40outlook.com), [#3 (Anthonin Bonnefoy, 2026-07-01)](https://www.postgresql.org/message-id/CAO6_Xqpg9t6n0iibUwqihfEx3xPwWtRFGenxi%3DtKWoCNqyCcGA%40mail.gmail.com)

**Question:** Should deferrable unique constraints count toward the one-row bound, and what behaviour do bulk loads inside the deferral window get?

Counter-check: held. The existing btcostestimate shortcut also ignores deferrability, but it only sets numIndexTuples. v3 extends the assumption to path rows, which affects join choice, and v2 and v3 disagree on this with no decision in the thread.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v3 adds a unique_path field to IndexPath in pathnodes.h and touches pathnode.h, which extensions that build index paths see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | Row estimates and plan choice change for full-key equality lookups on multi-column unique indexes. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A path-level row count that differs from the relation estimate is an exception later planner work must keep in mind. |
| performance on the hot path | **yes** | is_path_unique runs each time index paths are built, and the thread shows no planning-time measurement. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How do join size estimates, which use the relation-level row count, stay consistent with an index path that reports rows=1?
2. Why not clamp at relation level where relation_has_unique_index_for() applies, and keep a path-level clamp only for unique partial indexes?
3. Should deferrable unique constraints count toward the one-row bound, given that v2 excludes them and duplicates can exist until commit?
4. Do index AMs other than B-tree get the unique path behaviour, and through which code path?
5. What is the planning-time cost of is_path_unique on queries with many indexes, and do any existing regression plans change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows planner size estimation and index costing (costsize.c, selfuncs.c, indxpath.c). No such committer has posted in the three-message thread.
- **Liability:** Plan regressions from changed estimates on a common query pattern, in every later release. The owner also has to defend any exception to the equal-row-count rule for paths of one relation.
- **Author follow-up:** The author answered the review within three days with a reworked v3 and has moved the entry across three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Lookups by a composite unique key are common, and the reviewer in message #2 agrees the misestimate is real. v2 and v3 differ on the level of the clamp and on deferrable constraints, and the patch touches core planner costing files.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` New IndexPath field reuses the existing UniquePath name. Refuted (no failure scenario): The scenario depends on a hypothetical later patch misreading the field as a uniqueness proof. The field name is a code-level naming point, which is outside levels 1-2; the deferrable question is covered by standards-compat-3.
- `maintenance-fit-1` Path-level row clamp breaks equal rowcounts across paths of one relation. Refuted (not specific): Duplicate of standards-compat-1, which states the same README rule with a more concrete scenario.
- `maintenance-fit-2` Planner flag on IndexPath sidesteps the amcostestimate interface. Refuted (no failure scenario): amcostestimate already receives the IndexPath, so the flag reaches every AM through the existing interface, and AMs that call genericcostestimate get the selectivity change. B-tree is the only in-core AM with unique indexes, and the objection names no affected extension AM.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent is the same idea. P24's revert reason, bugs and possible planner slowdown, is generic and would apply to any planner change, so it does not support an objection here. The thread cites no earlier discussion of this idea.
- standards-compat: 3 objection(s). The proposal adds no syntax, GUC, catalog object or pg_dump state, so those touch points do not apply.
- replication-ondisk: 0 objection(s). The proposal changes only planner estimates and costing, with no on-disk, WAL, catalog, replication or wire protocol change. The new IndexPath field, and any create_index_path signature change in pathnode.h, is normal churn for a major version and the patch is not proposed for backpatching.
- security: 0 objection(s). The proposal changes only planner row estimates and costs for full-key equality lookups on unique indexes, and it adds no new privilege, object, function or exposed data. A wrong one-row estimate, for example from a deferrable constraint, affects plan quality only on the table owner's own data, so it is not a privilege or exposure issue.
- maintenance-fit: 2 objection(s). The fit with core is sound: composite unique-key lookups are a general case. No committer has engaged across three CommitFests, which affects committability only and is not counted as an objection.
- already-exists: 0 objection(s). No existing Postgres feature or extension gives a one-row estimate for full-key equality on a multi-column unique index. Extended statistics do not reliably produce that bound, and a planner hint extension only forces a plan without fixing the estimate. v3's is_path_unique extracts and generalizes the existing btcostestimate check, so it does not add a second mechanism. Whether it should sit beside relation_has_unique_index_for is a design question for another angle.
- demand: 0 objection(s). Demand holds up from this angle: lookups by a composite unique key are a common query pattern, and the only reviewer (message #2) agreed the misestimate is a real planner problem. The evidence is one synthetic skewed example with no field report, but the cost falls mainly on queries that match a full unique key, so a demand objection would not stand.

### Precedents retrieved

- P1: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P2: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Indexes with Included Columns (was Covering + unique indexes)](https://commitfest.postgresql.org/patch/961/) (cf, Returned with feedback, 2017-04-08) - judged not relevant
- P4: [Covering + unique indexes](https://commitfest.postgresql.org/patch/433/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P5: [Covering + unique indexes](https://commitfest.postgresql.org/patch/607/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P6: [Covering + unique indexes](https://commitfest.postgresql.org/patch/808/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P7: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P8: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P9: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Refactoring speculative insertion with unique indexes a little](https://commitfest.postgresql.org/patch/403/) (cf, Rejected, 2016-09-20) - judged not relevant
- P11: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P12: [Unique Joins](https://commitfest.postgresql.org/patch/129/) (cf, Returned with feedback, 2016-04-09) - judged not relevant
- P13: [Typos in the comment for the estimate_multivariate_ndistinct()](https://commitfest.postgresql.org/patch/5740/) (cf, Withdrawn, 2025-05-07) - judged not relevant; reason summarised from its thread
- P14: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant; reason summarised from its thread
- P15: [Adjust tuples estimate for appendrel](https://commitfest.postgresql.org/patch/4711/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P16: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P17: [Erase the distinctClause if the result is unique by definition](https://commitfest.postgresql.org/patch/2433/) (cf, Returned with feedback, 2022-04-05) - judged not relevant
- P18: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P19: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant
- P20: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P21: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P22: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P23: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P24: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - used in the review

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 119 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 72 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:09:56.228Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
