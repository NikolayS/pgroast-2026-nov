# Design pre-review: "variable not found in subplan target list", DISTINCT agg over Gather`

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7600](https://commitfest.postgresql.org/patch/7600/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [BUG #19804: "variable not found in subplan target list", DISTINCT agg over Gather`](https://www.postgresql.org/message-id/flat/19804-594c35d8647fd7ef%40postgresql.org), started 2026-10-28
- Messages read: 5 of 5
- Patch prose read: WIP-Supply-valid-input-Vars-for-sort-EC-selection.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real, reproducible on 16 through master, and the patch follows the direction suggested in message #3. Two fixable level 2 points survive, and neither argues against the fix. standards-compat-1 asks which working plans the stricter relation_can_be_sorted_early() check loses, and replication-ondisk-1 asks what the paths.h change means for back branches. Both need a committer who knows the planner to settle scope per branch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review cannot check the rules for independently supplied Vars, PHVs and nullingrels, or the Append and MergeAppend column-position checks. It also cannot check which paths.h declarations change.

## The proposal, restated

**Problem.** A query with a presorted DISTINCT or ORDER BY aggregate fails at plan time with "variable not found in subplan target list" when a Gather sits under the sort. The planner picks EC member b.y as computable because it appears inside the child output b.y * 2, but the child does not output b.y. The report reproduces this on 16 through master, while 14 returns no rows.

**Proposed design.** prepare_sort_from_pathkeys() gets an explicit list of Vars and quasi-Vars that the input independently supplies, and find_computable_ec_member() stays unchanged. Append and MergeAppend build and attach their children before sort columns are prepared, with column positions taken from the parent targetlist and checked against every child. relation_can_be_sorted_early() considers only Vars and quasi-Vars that are exposed on their own.

**User-visible surface.**
- prepare_sort_from_pathkeys() gets a changed signature for the explicit input list, visible to extensions that call it.
- One existing partition_join EXPLAIN output now shows an equivalent EC member as its sort key.

**What the author claims.**
- The patch fixes the reported query.
- All regression tests pass with assertions enabled.
- The relation_can_be_sorted_early() change is conservative and may reject some valid early-sort plans.
- v2 fixes compiler warnings that CI found in non-assert builds.

**Thread state.** Tom Lane found the cause, a hidden assumption in 9f954177, and suggested having the caller supply the input list. Tender Wang posted a WIP patch and v2, but nobody has reviewed either, and the author asks for feedback on local PHV handling.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Conservative early-sort check may change plans for working queries

fixable · level 2, rubric item 8 · planner behaviour change, backpatchability · `standards-compat-1`

A user on 16.x has a parallel query whose plan sorts early below a Gather, and the query does not hit the bug. After a minor upgrade, relation_can_be_sorted_early() rejects that sort, so the planner picks a slower plan. The user finds out from latency, with no error and no setting to undo it.

Message #4 says the new fallback 'might reject some otherwise valid early-sort opportunities', and no measure of the lost plans exists. The bug spans 16 through master, so a fix will likely be backpatched, and in a back branch a plan change for queries that are not broken is a compatibility cost. The proposal does not separate the narrow crash fix from the broader tightening, or say which branches each part should go to.

Thread: [#4 (Tender Wang, 2026-10-30)](https://www.postgresql.org/message-id/CAHewXN%3Dw6BjEHDfkC5ZTzc35FHHDc%3DKE8Nkw%3DYj_3_K-VvVRNw%40mail.gmail.com), [#5 (Tender Wang, 2026-11-09)](https://www.postgresql.org/message-id/CAHewXNn3dOxH9qf9yJrdrbBVxRy%3DkDMFjwdZ%2BvZOK6%3Dq-_PUzQ%40mail.gmail.com)

**Question:** Which queries that plan and run correctly today lose an early sort under the new check, and can that part stay out of the back branches?

Counter-check: held. Message #4 itself says the new fallback may reject valid early-sort plans, and the thread gives no example or count. Message #3 expected a change in this function but did not settle its scope or the target branches.

### 2. Back-branch ABI effect of the paths.h change is not stated

fixable · level 2, rubric item 11 · backpatchability, ABI / extension API · `replication-ondisk-1`

The fix goes into a 16.x or 17.x minor release with a changed declaration in paths.h. An extension built against the earlier minor release keeps calling the old signature. It may crash or misplan after the server is updated in place, with no rebuild step to warn the user.

The bug exists on 16 through master (message #1), so the fix needs back-patching, but the patch prose does not name target branches. The patch touches src/include/optimizer/paths.h, and the brief lists a signature change that extensions can see. The commit message keeps find_computable_ec_member() unchanged but does not say which exported declaration changes. A committer would likely ask for a separate back-branch variant that keeps existing signatures, for example a new function beside the old one.

Thread: [#1 (PG Bug reporting form, 2026-10-28)](https://www.postgresql.org/message-id/19804-594c35d8647fd7ef%40postgresql.org), [#4 (Tender Wang, 2026-10-30)](https://www.postgresql.org/message-id/CAHewXN%3Dw6BjEHDfkC5ZTzc35FHHDc%3DKE8Nkw%3DYj_3_K-VvVRNw%40mail.gmail.com), [#5 (Tender Wang, 2026-11-09)](https://www.postgresql.org/message-id/CAHewXNn3dOxH9qf9yJrdrbBVxRy%3DkDMFjwdZ%2BvZOK6%3Dq-_PUzQ%40mail.gmail.com)

**Question:** Which exported declarations in paths.h change, and what form of the fix keeps them ABI-compatible on 16 through 19?

Counter-check: held. The patch touches paths.h and names no target branches, so the back-branch question stands. prepare_sort_from_pathkeys() is static in createplan.c, so the concern covers only the paths.h declarations, and a purely added function would cause no ABI break.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch touches src/include/optimizer/paths.h, a header that extensions include. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch changes sort EC member choice, Append and MergeAppend plan creation, and the relation_can_be_sorted_early() check. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug exists on 16 through master, and the patch names no target branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | createplan.c gains rules for which child outputs count as independently supplied Vars, PHVs and nestloop parameters. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which queries that plan and run correctly today lose an early sort under the new relation_can_be_sorted_early() check?
2. Which branches does the patch target, and does the stricter relation_can_be_sorted_early() check go to the back branches or only to master?
3. Which declarations in paths.h change, and does the back-branch form of the fix keep existing exported signatures?
4. Why is the output-side nullingrels rule for outer-join inputs correct for choosing a sort EC member?
5. Which local PHVs does the input list keep, and what case shows that each kept PHV is available to the projection?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane diagnosed the cause in message #3 and suggested the caller-supplied input list. No committer has reviewed the WIP patch or v2 yet.
- **Liability:** The owner signs for planner behaviour on up to five branches: the new input-list rules in createplan.c, the Append and MergeAppend reordering, and any plan changes in minor releases.
- **Author follow-up:** The author posted a WIP patch and then v2 to fix CI compiler warnings, and asks for feedback on local PHV handling.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** A valid query fails with an internal error on all supported branches from 16 up, though the trigger needs a rare plan shape found by a fuzzer. The patch is about 35 KiB in createplan.c and equivclass.c, and review needs someone who knows EC and setrefs behaviour.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Header change in paths.h has no stated plan for back branches. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same paths.h concern and adds that the commit message does not name the changed declaration.
- `maintenance-fit-1` Signature change to prepare_sort_from_pathkeys() conflicts with a back-patched fix. Refuted (wrong facts): prepare_sort_from_pathkeys() is a static function in createplan.c, so no extension can call it and its signature can change in a minor release. The remaining paths.h concern is covered by replication-ondisk-1.
- `maintenance-fit-2` Stricter early-sort check may lose plans for unaffected queries. Refuted (not specific): Duplicate of standards-compat-1, which makes the same point and also asks to separate the crash fix from the stricter check per branch.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The precedent search returned no row about the same or a similar idea. The thread names commit 9f954177 (messages #2 and #3) as the change that introduced the hidden assumption. That is a source of the bug, not an earlier rejected attempt, and the patch keeps its behaviour on purpose. From the prior-attempts angle, no recorded rejection reason applies.
- standards-compat: 2 objection(s). SQL syntax, GUCs, pg_dump and psql are not affected. The only partition_join EXPLAIN change names an equivalent sort key, which seems acceptable on master.
- replication-ondisk: 1 objection(s). The fix stays in the planner and adds no on-disk, WAL, catalog or wire-protocol state. Plans are not persisted, so pg_upgrade and standbys are not affected. The only concern from this angle is back-branch ABI.
- security: 0 objection(s). The fix stays inside the planner and changes how sort columns are chosen. It adds no privilege, role, GUC, catalog object or visible data, and it lets no unprivileged user see or do anything new. The query in the report errors out at plan time, so there is no wrong-result leak either. Leakproofness and RLS quals are not involved in choosing EC members for the sort.
- maintenance-fit: 2 objection(s). The fix itself is wanted and follows the direction suggested in message #3. The objections cover back-branch cost and side effects, and neither argues against fixing the bug.
- already-exists: 0 objection(s). This is a fix for a planner internal error on 16 through master. No existing feature or extension covers it. Setting enable_presorted_aggregate = off avoids the error for a user, but it does not fix the planner, so the already-implemented angle has nothing to raise.
- demand: 0 objection(s). This is a bug fix for a plan-time internal error on supported branches 16 through master, which 14 did not have. The bug comes from a fuzzer report with a reproducer, and a committer confirmed the cause in message #3. The demand angle does not apply: the users are anyone whose valid query fails, and a correct-results regression needs no proof of demand.

### Precedents retrieved

- P1: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P2: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant; reason summarised from its thread
- P3: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant; reason summarised from its thread
- P4: [Don't shut down Gather\[Merge\] early under Limit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=080313f8296fb0bcc74bd70fc8e15cd64f45945e) (revert, Partially reverted, 2019-11-26) - judged not relevant
- P5: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant; reason summarised from its thread
- P6: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant; reason summarised from its thread
- P7: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P8: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P9: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - judged not relevant; reason summarised from its thread
- P10: [Use wait_classes.h in pg_prewarm](https://commitfest.postgresql.org/patch/6946/) (cf, Withdrawn, 2026-10-05) - judged not relevant; reason summarised from its thread
- P11: [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://commitfest.postgresql.org/patch/7034/) (cf, Returned with feedback, 2026-09-10) - judged not relevant
- P12: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P13: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P14: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant
- P15: [Fix comment about generate_gather_paths](https://commitfest.postgresql.org/patch/2876/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P16: [Add support for Distinct in Window aggregates](https://commitfest.postgresql.org/patch/4243/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P17: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P18: [new diagnostic items for the dynamic sql](https://commitfest.postgresql.org/patch/3537/) (cf, Returned with feedback, 2022-12-14) - judged not relevant
- P19: [Fix the statistics gathering in case of commit and chain](https://commitfest.postgresql.org/patch/3657/) (cf, Returned with feedback, 2022-10-06) - judged not relevant
- P20: [Early locking option to parallel backup](https://commitfest.postgresql.org/patch/1367/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P21: [Support target_session_attrs=read-only and eliminate the added round-trip to detect session status.](https://commitfest.postgresql.org/patch/1148/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P22: [\[PoC\] Implementation of distinct in Window Aggregates: take two](https://commitfest.postgresql.org/patch/4519/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P23: [Add a reset_computed_values function in pg_stat_statements](https://commitfest.postgresql.org/patch/2275/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P24: [brin: avoid errors processing work item following concurrent reindex](https://commitfest.postgresql.org/patch/2838/) (cf, Returned with feedback, 2021-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 131 s; cost reported by the CLI: $1.64
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:04:49.551Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
