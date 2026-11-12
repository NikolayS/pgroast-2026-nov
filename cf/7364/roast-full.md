# Design pre-review: Costing for parallel scans with few/single row produced in the outer side

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7364](https://commitfest.postgresql.org/patch/7364/) (Needs review); authors: Matthias van de Meent (mmeent)
- Thread: [Costing for parallel scans with few/single row produced in the outer side](https://www.postgresql.org/message-id/flat/CAEze2Wi%2BJp26mhBJvrMshKGYNTtPcxzd1BWoEW_7%2BeH6saj3gQ%40mail.gmail.com), started 2026-09-28
- Messages read: 6 of 6
- Patch prose read: v2-0001-optimizer-Improve-costing-for-low-rowcount-parall.patch, v1-0001-optimizer-Better-costing-for-low-rowcount-paralle.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is stated and a committer who knows parallel costing treats it as real, so thin demand (demand-1) alone does not stop review. The design does: effective_workers has no defined invariant (maintenance-fit-1), and v2 still costs 5 outer rows below 4 at 4 workers (standards-compat-1). Message #6 also reports wrong values for partial aggregates and Parallel Append, which the patch does not target (demand-2). The list needs to agree on the invariant before code review makes sense.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the value of effective_workers for paths that set only parallel_workers and the planning time cost of the change could not be checked. Messages #4 to #6 were read only through the objections that cite them.

## The proposal, restated

**Problem.** When the outer side of a parallel plan produces fewer rows than there are workers, the planner still divides the cost of inner join nodes by about the worker count. Only one worker finds the row and runs the joins, so the planner picks a parallel seq scan over a cheaper primary key index scan. The author hit this in a TPCC benchmark on pg18 and master.

**Proposed design.** The patch adds Path->effective_workers, an estimate of how many workers can do useful work given the row estimate, and uses it in place of the planned worker count when it computes the parallel divisor. Pathnode code propagates the value through joins, upper rels and, in v2, Append. v2 makes the Path worker fields int16 so that Path does not grow.

**User-visible surface.**
- New Path field effective_workers in pathnodes.h, visible to planner extensions and custom path providers.
- Path worker count fields change type from int to int16 in v2.
- Changed signatures in optimizer/pathnode.h and paths.h, including create_append_path, which may affect extensions.

**What the author claims.**
- Parallelism adds nothing past the outer node when the outer side yields one row, so inner nodes should get no worker discount.
- With the patch the TPCC reproducer, about 19 GB, uses a non-parallel PK scan on district.
- Only queries that underestimate rows below the useful worker count can plan worse, and those plans already have bigger problems.
- int16 worker fields are safe because MAX_PARALLEL_WORKER_LIMIT is 1024.
- v2 fixes the negative effective_workers underflow that Haibo Yan reported.

**Thread state.** Robert Haas doubts that the number of reports justifies a bigger Path, and notes that the correction stops at rows >= workers. Haibo Yan reports open problems in v2: wrong caps for partial aggregates, wrong Append math, and lower cost at higher row counts; he asks for a defined invariant. CFbot says the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Every path constructor must now set effective_workers, and nothing checks it

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change, ABI / extension API · `maintenance-fit-1`

A later patch adds a new partial path type, or a custom scan provider builds partial paths, and copies parallel_workers without deriving effective_workers. The planner then computes a wrong parallel divisor and wrong Gather row counts. Nobody sees this until a user reports a bad plan, because nothing errors and the regression suite tests few such shapes.

v1 touched four files and v2 touched ten, because the value has to be propagated through joins, upper rels, set operations and Append. Message #4 and message #6 found wrong values in partial aggregates, partial DISTINCT and Append, each a place where propagation was wrong without any test failing. Message #6 notes the field has no defined invariant, so a future author has no rule to follow when adding a node type.

Thread: [#4 (Haibo Yan, 2026-09-30)](https://www.postgresql.org/message-id/CABXr29FUZJsb1xHbzY_5%3DC2nuvbCUXWCNmBKzZXTgjkFqzdKAQ%40mail.gmail.com), [#5 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-04)](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com)

**Question:** What written invariant for effective_workers lets a future path constructor or extension set it correctly, and what would fail if it set it wrong?

Counter-check: held. Messages #4 and #6 found wrong values in partial aggregates and Append with no test failing, and message #6 asks for an invariant that the author has not yet given. The file count grew from four to ten between v1 and v2.

### 2. Plans for existing parallel queries change, with cost falling as estimates rise

fixable · level 2, rubric item 8 · planner behaviour change, backpatchability · `standards-compat-1`

After upgrade, autoanalyze moves a join's outer estimate between 4 and 5 rows at 4 workers. The cheaper plan flips each time, because v2 costs R=5 below R=4. The DBA sees unstable plans with no setting change and no switch to get the old costing back.

Message #6 shows the R=4 to R=5 cost drop is still present in v2. Message #5 attributes the drop to the existing leader heuristic. Message #6 counters that effective_workers makes that heuristic depend on cardinality at a fixed worker count, so the effect is new. Message #3 also accepts that queries which underestimate below the worker count can plan worse. The proposal gives users no way to opt out or bound that change.

Thread: [#3 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-04)](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com)

**Question:** Can the cost be made non-decreasing in estimated rows, and what do users with underestimated queries do if their parallel plans disappear after upgrade?

Counter-check: held. Message #6 measures a lower cost at R=5 than at R=4 in v2, and the author's reply in message #5 does not remove it. The lack of an opt-out setting is common for planner changes and adds no weight.

### 3. Partial paths built by extensions get no defined effective_workers

fixable · level 2, rubric item 9 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

A custom scan provider or FDW builds a partial path and sets parallel_workers, as it does today, but has no reason to set the new field. In messages #4 and #6, effective_workers = 0 gives a divisor of 1, so such a path would be costed with no worker discount and lose to core paths. Users would see only a plan change after a major upgrade.

v2 adds Path->effective_workers and makes get_parallel_divisor() depend on it, so out-of-core code that creates partial paths now has a new duty. The proposal does not say what value a path gets when its creator leaves the field unset, and nothing fails at build time or run time when it is missing. v2 also changes create_append_path and other signatures in pathnode.h and paths.h. Those changes break the build loudly, but a missing field assignment does not.

Thread: [#4 (Haibo Yan, 2026-09-30)](https://www.postgresql.org/message-id/CABXr29FUZJsb1xHbzY_5%3DC2nuvbCUXWCNmBKzZXTgjkFqzdKAQ%40mail.gmail.com), [#5 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-04)](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com)

**Question:** What effective_workers does a partial path get when an extension sets only parallel_workers, and how would such an extension learn it must change?

Counter-check: held. The proposal does not say what value a path gets when its creator sets only parallel_workers. Messages #4 and #6 show that a value of 0 gives a divisor of 1, so the silent plan change is plausible, though only the code can confirm it.

### 4. Gain is narrow while plan and API changes reach everyone

fixable · level 1b, rubric item 4 · desirability, ABI / extension API, planner behaviour change, maintenance burden ("who maintains this in 10 years") · `demand-2`

A user with a one-group partial aggregate or a Parallel Append over low-row partitions gets changed Gather row estimates and possibly a worse plan after upgrade. Extensions that call create_append_path stop compiling, though they never had the single-row outer case.

The benefit applies only when the estimated outer rows fall below the worker count (message #3). v2 changes create_append_path and other signatures, and Haibo Yan shows wrong estimates for partial aggregates and Append, queries the patch does not target (message #6). So the cost falls on all parallel plans and on extensions, while the gain stays with a small set of queries.

Thread: [#3 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6 (Haibo Yan, 2026-10-04)](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com)

**Question:** Can the change be limited to the join paths above a near-single-row parallel scan, so partial aggregates, Append and extension APIs see no change?

Counter-check: held. Message #6 shows wrong Gather row estimates for one-group partial aggregates and Append, which the patch does not target, and the author has not answered. The extension compile break must not be used as support, because signature changes at major versions are normal.

### 5. Demand rests on one benchmark and one older report

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds a new Path field that every future path type must set correctly. Few production users ever hit the single-row outer case it targets, so later planner patches carry the cost of keeping it right for little real gain.

The trigger is a TPCC benchmark at one vendor (message #1). Robert Haas searched and found one clear matching report; the other two cases he found fall outside the patch (messages #2, #3). He said the bigger Path is hard to justify "as things are" (message #2). The int16 change in v2 removes the size cost, but the semantic and maintenance cost stays.

Thread: [#1 (Matthias van de Meent, 2026-09-28)](https://www.postgresql.org/message-id/CAEze2Wi%2BJp26mhBJvrMshKGYNTtPcxzd1BWoEW_7%2BeH6saj3gQ%40mail.gmail.com), [#2 (Robert Haas, 2026-09-29)](https://www.postgresql.org/message-id/CA%2BTgmobUaa0OAkUBOy0E-BxkG1k%2B0FZacFzp93_sPJPDQgx3Tw%40mail.gmail.com), [#3 (Matthias van de Meent, 2026-09-29)](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5 (Matthias van de Meent, 2026-10-02)](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com)

**Question:** Which production workloads, besides the TPCC benchmark and the one older list report, get a parallel seq scan over a cheaper PK lookup because of this discount?

Counter-check: held. Message #2 reports one clear matching earlier report, and the author agrees in message #3 that the other two cases fall outside the patch. An engaged committer who sees the problem as real lowers the weight, but the thread shows no further demand.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v2 adds Path->effective_workers, narrows the worker fields to int16 and changes create_append_path and other signatures in pathnode.h and paths.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The parallel divisor now depends on estimated rows, so costs and plans of existing parallel queries change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix and the problem was seen on pg18, but the Path layout and signature changes cannot go into a stable branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every partial path constructor, in core and in extensions, must set effective_workers correctly, and nothing fails when it does not. |
| performance on the hot path | **yes** | The patch changes the Path struct and adds work per path, and no planning time or memory measurement is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Demand rests on one benchmark and one older report |

## 5. Questions the author must answer before anyone reviews code

1. What invariant does effective_workers satisfy: useful workers, leader slots, or worker occupancy?
2. Can the cost be made non-decreasing in estimated outer rows at a fixed worker count, so that 5 rows never cost less than 4 at 4 workers?
3. How should effective_workers be computed for partial aggregates, partial DISTINCT and Parallel Append so that the Gather row estimates reported in message #6 stay correct?
4. What effective_workers does a partial path get when an extension sets only parallel_workers, and what fails if the value is wrong?
5. Which workloads, besides the TPCC benchmark and the one older report found in message #2, pick a parallel seq scan over a cheaper PK lookup because of this discount?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas takes part in the thread (message #2) and reviews the approach, but he says he is not sure what is best. The owner must know parallel query costing and partial path generation.
- **Liability:** The owner signs for the cost model of every parallel plan: plan changes after upgrade, and correct effective_workers values in every current and future partial path type, with no end date.
- **Author follow-up:** The author answered the first review within a day and posted v2 for the reported underflow. The v2 findings in message #6 and the CFbot rebase request from 2026-10-08 have no visible reply.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The gain applies only when estimated outer rows fall below the worker count, and message #2 found one clear earlier report. v2 is about 42 KiB across ten optimizer files and still gives wrong values for Append and partial aggregates, so the meaning of the field must be settled first.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Path field and pathnode signature changes leave extension contract undefined. Refuted (not specific): Duplicate of replication-ondisk-1, which states the unset-field case more concretely. Changed signatures in pathnode.h between major versions are normal practice and extensions adapt to them at each release.
- `maintenance-fit-2` Public Path and pathnode API change for one reported workload. Refuted (not specific): Duplicate of demand-1 on the demand point. The rest does not hold: compile breaks at a major version are accepted, and C promotes int16 operands to int, so sums of values capped at 1024 do not overflow.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed capping the parallel divisor by an estimate of useful workers. [P1] and [P2] covered partial UNION ALL and subquery costing, and [P6] covered a parallel_workers reloption, so their recorded reasons do not apply here. Message #2 points to earlier user reports of similar plans, not to earlier patches, so none of them is a prior rejection.
- standards-compat: 2 objection(s). The change adds no SQL syntax, GUC, catalog entry or dump state. The objections cover plan stability and the extension-facing Path contract only.
- replication-ondisk: 1 objection(s). The patch is planner-only and changes no on-disk format, WAL, catalog or wire protocol. The only cross-boundary state is the extension-facing Path API covered above.
- security: 0 objection(s). The patch changes only planner cost estimates for parallel paths. It adds no privilege, object, function, setting or monitoring output, and it exposes no new data. Plan choice is already user-influenced through statistics and settings, so the change opens no new path to escalation or denial of service.
- maintenance-fit: 2 objection(s). Two objections within the maintenance angle. Robert Haas, who knows parallel query costing, is engaged in the thread, which lowers the risk under item 16.
- already-exists: 0 objection(s). Existing workarounds do not meet the need. `alter table district set (parallel_workers = 0)`, a lower max_parallel_workers_per_gather, or a higher parallel_setup_cost can avoid the bad plan, but each turns off parallelism for every query on that table or session, and none fixes the inner-node discount in the cost model. No extension or hook can change get_parallel_divisor for core join paths, so the gap the patch addresses has no alternative outside core. The design concerns (meaning of effective_workers, Append math, Path size) are for other angles.
- demand: 2 objection(s). Demand objections are listed above. The listed precedents are about other parallel costing changes and do not share this proposal's reason for rejection.

### Precedents retrieved

- P1: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P2: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - used in the review; reason summarised from its thread
- P3: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant; reason summarised from its thread
- P4: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P5: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - used in the review; reason summarised from its thread
- P7: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant; reason summarised from its thread
- P8: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P9: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P10: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P11: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Revise get_cheapest_parallel_safe_total_inner](https://commitfest.postgresql.org/patch/4285/) (cf, Returned with feedback, 2026-02-09) - judged not relevant
- P13: [support tid range scan in parallel](https://commitfest.postgresql.org/patch/4982/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [The Prepare statement with the skewed parameter uses the custom plan](https://commitfest.postgresql.org/patch/4839/) (cf, Withdrawn, 2024-09-22) - judged not relevant
- P15: [Parallel leader process info in EXPLAIN output](https://commitfest.postgresql.org/patch/2315/) (cf, Returned with feedback, 2020-03-17) - judged not relevant
- P16: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P17: [Bloom index cost model](https://commitfest.postgresql.org/patch/2043/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P18: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P19: [Disable parallel query by default](https://commitfest.postgresql.org/patch/5751/) (cf, Rejected, 2026-02-11) - judged not relevant
- P20: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant
- P21: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P22: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P24: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 115 s; cost reported by the CLI: $1.44
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 81 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:45:10.081Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
