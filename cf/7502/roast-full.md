# Design pre-review: Fix rescan costing of single-batch hash joins

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7502](https://commitfest.postgresql.org/patch/7502/) (Needs review); authors: Baji Shaik (bajishaik)
- Thread: [Fix rescan costing of single-batch hash joins](https://www.postgresql.org/message-id/flat/CA%2Bfm-RMQd9rvbwQD4aL4iAYSo3869DPH8_CX%2BBGBCS91aKQ-og%40mail.gmail.com), started 2026-10-21
- Messages read: 5 of 5
- Patch prose read: 0001-Fix-rescan-costing-of-single-batch-hash-joins.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived at levels 1a, 1b or 2. The problem is stated with two reproducible scripts, and Etsuro Fujita agreed in message #2 that the costing does not match ExecReScanHashJoin(). The confidence is low because the panel raised no objections and the patch code was not read, so the result rests on the prose alone.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked how the outer rescan charge and the parameterized-inner charge combine, or whether parallel and multi-batch hash joins are affected. The thread reports no plan-change results beyond the regression suite and the two scripts.

## The proposal, restated

**Problem.** cost_rescan() charges nothing on rescan for a single-batch hash join, though ExecReScanHashJoin() reruns the outer side and rebuilds the hash table when inner parameters change. A hash join over a LATERAL subquery with sorting or grouping is costed as if that work ran once, so the planner picks slow plans. With plan caching, custom plans also look cheaper than generic plans, so generic plans are never chosen.

**Proposed design.** On rescan, cost_rescan() charges the outer path's own rescan cost instead of zero. When the inner path is parameterized, it charges the full join cost, because the hash table is rebuilt every loop. The change is in costsize.c only, with a new regression test in join.sql.

**What the author claims.**
- With 100 groups and a 10k-row table, master picks a hash join at 43-60 ms and the patch picks nested loop plus Memoize at about 1.5 ms.
- With a 1M-row lookup table both builds pick the same plan.
- The underestimate also appears with GROUP BY, DISTINCT, ORDER BY without LIMIT, and inlined SQL set-returning functions taking an outer column.
- Since SQL functions use the plan cache in 18, the underestimate keeps prepared statements and SQL functions on custom plans (19 ms per call versus 0.7 ms generic).
- No existing regression plans change; make check, contrib and isolation tests pass; master only.

**Thread state.** Etsuro Fujita agreed the costing does not match the executor and that the fix is master-only, and plans to review next month. Nazneen Jafri reproduced both scripts on 18.4 builds; no review of the code has been posted.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| planner behaviour change | **yes** | cost_rescan() in costsize.c charges more for rescans of single-batch hash joins, so nested loop plans over hash joins can change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but author and reviewer agree it is master only because it changes plans. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. When the inner path is parameterized, does the full join cost replace the outer path's rescan cost, or can both charges apply to one join?
2. Which plans change on a join-heavy benchmark such as TPC-H or the join order benchmark, and do any of them run slower with the patch?
3. Does the new charge apply to parallel hash joins and to multi-batch hash joins, or do those cases keep their current rescan cost?
4. How many prepared statements or SQL functions that are fast on custom plans today would move to a slower generic plan with the higher custom plan cost?
5. Does merge join or another node handled by cost_rescan() have the same underestimate when its input reruns on every rescan?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Etsuro Fujita agreed with the diagnosis in message #2 and signed up as reviewer; the owner must know planner costing and the hash join rescan path in the executor.
- **Liability:** The owner signs for plan changes in nested loops over hash joins after the next major upgrade, and for reports of queries that become slower, with no backpatch to limit the exposure.
- **Author follow-up:** The author answered the next day and added the CommitFest entry when asked (message #3); no later sign either way in the visible messages.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 6.2 KB, touches costsize.c and one regression test, and adds no user-visible surface. It affects LATERAL queries and SQL functions on the plan cache in 18, and a planner-area committer plans to review it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed this idea. [P1] and [P4] were planner costing heuristics that failed on counterexamples or averaged alternatives, and this patch aligns cost_rescan() with ExecReScanHashJoin(). Message #1 cites the original cost_rescan() commit and the multi-batch fix, which kept the old assumption, and calls Andrei's Aggregate report related but separate; the thread records no rejection of this idea.
- standards-compat: 0 objection(s). The patch adds no syntax, keyword, GUC, catalog object, psql change or dump format change, so there is no user-facing contract to check against the standard or pg_dump. The only compatibility effect is that some plans change after a major upgrade, which is normal for a master-only costing fix. Whether those plan changes regress other workloads, including more prepared statements moving to generic plans, belongs to the performance angle.
- replication-ondisk: 0 objection(s). The patch changes only cost estimation in costsize.c plus a regression test. It touches no on-disk format, WAL, catalog, wire protocol or exported struct, so pg_upgrade, standbys and subscribers are unaffected. Plan choices may differ between primary and standby after an upgrade, but that is normal planner behaviour and not persistent state.
- security: 0 objection(s). The patch changes only planner cost estimates in costsize.c for hash join rescans. It adds no new privilege, object, setting, function or view, and changes no access-control path. A plan choice exposes no data that the user could not already see through EXPLAIN on their own query. No security or privilege objection applies.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch changes one cost function in costsize.c to match what ExecReScanHashJoin() already does. It adds no new GUC, catalog, API or dependency, and it adds no per-tuple or per-query cost outside planning. It belongs in core, and a committer who knows the planner, Etsuro Fujita, has agreed with the diagnosis and signed up to review it. Two points could not be checked without the code or more data. The material does not show whether the outer rescan charge and the parameterized-inner charge can both apply to one join, and it reports no plan-change results beyond the regression suite. Those belong to the design and performance angles.
- already-exists: 0 objection(s). The proposal fixes a cost estimate that does not match what ExecReScanHashJoin() does. No existing feature or extension corrects cost_rescan(). Workarounds such as plan_cache_mode = force_generic_plan or enable_hashjoin = off only hide the problem per session or per query, and they do not help the plain-query case in repro_plain.sql.
- demand: 0 objection(s). No demand objection. This fixes planner costing that does not match what ExecReScanHashJoin() does, and it came from a real performance case, which a second person reproduced on 18.4 (message #4). Etsuro Fujita agreed the costing is wrong (message #2). The affected group, LATERAL queries and SQL functions on the plan cache in 18, is broad enough. Plan regressions for other users belong to the design angle.

### Precedents retrieved

- P1: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - used in the review; reason summarised from its thread
- P2: [Hashjoin fall back to block nested hashloop join](https://commitfest.postgresql.org/patch/2588/) (cf, Withdrawn, 2020-11-10) - judged not relevant; reason summarised from its thread
- P3: [Autovacuum dynamic cost_limit, cost_delay](https://commitfest.postgresql.org/patch/2983/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - used in the review; reason summarised from its thread
- P5: [Better estimate for filter cost](https://commitfest.postgresql.org/patch/1296/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P6: [hash joins with bloom filters](https://commitfest.postgresql.org/patch/1540/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P7: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P8: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P9: [cost_sort improvement](https://commitfest.postgresql.org/patch/1706/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P10: [Support run-time partition pruning for hash join](https://commitfest.postgresql.org/patch/4512/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P11: [bloom filter in hash joins with batching](https://commitfest.postgresql.org/patch/446/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P12: [accounting for memory used for BufFile during hash joins](https://commitfest.postgresql.org/patch/2109/) (cf, Returned with feedback, 2019-11-26) - judged not relevant
- P13: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P14: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant
- P15: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant
- P16: [Add information about generic/custom plan to pg_stat_statements](https://commitfest.postgresql.org/patch/2713/) (cf, Withdrawn, 2021-04-05) - judged not relevant
- P17: [The Prepare statement with the skewed parameter uses the custom plan](https://commitfest.postgresql.org/patch/4839/) (cf, Withdrawn, 2024-09-22) - judged not relevant
- P18: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P19: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P20: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P21: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P22: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P24: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 69 s; cost reported by the CLI: $1.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:02:00.587Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
