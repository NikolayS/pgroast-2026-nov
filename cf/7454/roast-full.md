# Design pre-review: Avoid O(N^3) planning time in choose_plan_name()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7454](https://commitfest.postgresql.org/patch/7454/) (Needs review); authors: Manuel Reyes Bravo (manu15)
- Thread: [O(N^3) planning time in choose_plan_name() since 8c49a484e8e](https://www.postgresql.org/message-id/flat/179161574267.1707709.13345376006450096249%40gmail.com), started 2026-10-10
- Messages read: 17 of 17
- Patch prose read: v6-0001-Avoid-O-N-3-planning-time-in-choose_plan_name.patch, measurements.txt, v1-0001-Avoid-O-N-3-planning-time-in-choose_plan_name.patch, repro.sql.txt (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Nothing that survived questions the goal or the design. The patch fixes a planning-time regression from 8c49a484e8e, and v6 implements the threshold hash table that Robert Haas proposed. The one surviving objection, maintenance-fit-1, asks for a lasting regression test that crosses the 32-name threshold, which is a fixable point. A committer who knows the planner must still decide on v6 and on whether it goes into REL_19_STABLE.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody could check that the hash path and the list path choose the same name in every collision case, or that v6 never truncates its keys. The memory use of the hash table is also unmeasured.

## The proposal, restated

**Problem.** Since commit 8c49a484e8e, choose_plan_name() tries suffixes 1, 2, 3, ... and scans every assigned name for each try, so planning is O(N^3) in subplans sharing a base name. A 4000-arm UNION plans in about 17 s on master against about 19 ms on v18. The author says only generated SQL hits this: many UNION arms, scalar sublinks or ORed IN subqueries.

**Proposed design.** v6 keeps the existing list and, once a statement has 32 subplan names, also puts them in a simplehash keyed by the C string. Each entry records whether the exact name is taken and the suffix where the search for a free name_N starts. Below 32 names the code runs as before, and the chosen names do not change.

**What the author claims.**
- v6 plans a 4000-arm UNION in about 22 ms, against 16.6 s on master and 19 ms on 18.6.
- 4000 CTEs with different names plan in 126 ms with v6, against 153 ms on master and 128 ms on 18.6.
- Generated names are byte-identical to master's in regression tests, four contrib suites and 32 collision statements.
- Small statements plan 2-4% slower with default builds, but equal with -falign-functions=64, so the author attributes the gap to code layout.
- Keeping today's names matters because users write them into pg_plan_advice strings, and pg_plan_advice ships in 19.

**Thread state.** Andres Freund and Tom Lane questioned the complexity and suggested a single counter; Tom found v3 hard to reason about. Robert Haas proposed the threshold hash table that v6 implements, and later asked why the UNION case is so much worse, which the author answered; no one has reviewed v6 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Two lookup paths must agree, and only rare statements use the hash

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a subplan kind or changes the naming rule, and updates the list search but not the hash entries. The regression suite has only small statements, so it stays green. A user whose generated SQL passes 32 names finds that their pg_plan_advice strings now point at different subplans.

v6 keeps the list and adds a simplehash above 32 names, so two structures must give the same name for every input (message #15). The author checked this with the threshold set to 0, but as a one-time run, and the prose does not say the suite keeps doing so. Names matter because advice strings refer to them (messages #12, #14). The list of touched files shows only subselect.sql, with no statement known to cross the threshold.

Thread: [#12 (Robert Haas, 2026-11-03)](https://www.postgresql.org/message-id/CA%2BTgmoZx7rAWBMMEjUnVhvqy8YsRarK8cfouw4DeiRHdgVbuxg%40mail.gmail.com), [#14 (Robert Haas, 2026-11-03)](https://www.postgresql.org/message-id/CA%2BTgmoZ6GiKx_tFoT3RDR3TBFt3cPeOjonTB9mi29Sj1ZpK_7Q%40mail.gmail.com), [#15 (Manu, 2026-11-04)](https://www.postgresql.org/message-id/179382931036.1743405.1568941311065599844%40gmail.com)

**Question:** Which regression test crosses the 32-name threshold with mixed generated and user-written names, so that a future change to one path alone fails the suite?

Counter-check: held. The two-path design copies ec_derives_hash and Robert Haas proposed it, so the design stands. But the visible prose names no subselect.sql statement that passes 32 names, and the threshold-0 check was a one-time run, so a later divergence in choose_plan_name() could pass the suite.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds fields to PlannerGlobal in src/include/nodes/pathnodes.h, which extensions that include that header see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | choose_plan_name() in planner.c changes how it finds a free subplan name; the author reports the chosen names stay the same. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The regression is in 19 and no reviewer has said whether v6 goes into REL_19_STABLE before 19.0. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The list search and the hash table must give the same name for every input from now on. |
| performance on the hot path | **yes** | Every statement with subplans calls choose_plan_name(); measurements.txt shows 2-4% slower small statements with default builds, equal with aligned builds. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which regression test crosses the 32-name threshold with mixed generated and user-written names, so that a later change to one lookup path alone fails the suite?
2. Is v6 meant for REL_19_STABLE before 19.0, or for master only?
3. How much planner memory does the hash table use for a statement with 4000 subplan names, compared with master?
4. Does the 32-name threshold hold for statements that mix generated and user-written names, beyond the two pure cases in measurements.txt section 3?
5. Should 19 keep the smallest-free-suffix naming, or switch to a single counter before release, given that pg_plan_advice strings refer to these names?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas proposed the threshold hash table that v6 implements, and Tom Lane and Andres Freund commented on earlier versions. No one has reviewed v6 yet.
- **Liability:** The owner signs for subplan names staying identical between the list path and the hash path, for as long as pg_plan_advice strings refer to those names.
- **Author follow-up:** The author produced six versions and detailed measurements in about four weeks, and answered reviewers' questions in the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only generated SQL with many same-named subplans hits the problem, but it is a regression from v18 of up to about 1000 times in planning time. The patch is 14.2 KB, mostly in planner.c, and reuses the simplehash threshold pattern of ec_derives_hash.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P4] was rejected because one static cache helped a single call pattern, but v6 uses a per-statement hash table that also covers different user-given names, as Robert asked in message #14. Message #8 refers to an earlier idea from Heikki that is not in the material, and message #12 reports it is not in this thread either.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, dump content or psql change, and v6 keeps master's subplan names byte-identical, so pg_plan_advice strings and EXPLAIN output stay the same. The one open contract question is whether 19 should switch naming schemes before release (messages #8, #12); that choice exists with or without this patch, and this patch does not decide it.
- replication-ondisk: 0 objection(s). The patch adds no persistent or cross-node state: no on-disk, WAL, catalog or protocol change. It adds a planner-local hash table and fields to PlannerGlobal in pathnodes.h, which are not ABI concerns before 19.0, and the chosen names stay the same, so stored pg_plan_advice strings keep matching.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes how the planner picks internal subplan names, adds no new object, setting, function or privilege, and keeps the names users see in EXPLAIN and pg_plan_advice unchanged. On denial of service, it removes an existing problem: today any user who can send a query can make a backend spend seconds of CPU in choose_plan_name() with a few thousand UNION arms, and v6 brings that back to v18 levels. The hash table grows with the number of subplans, so its size follows the length of the query text. v1 could loop forever in a non-assert build, ignoring cancel, when C code set aliases longer than NAMEDATALEN (message #5). v6 keys the table by the full C string, which removes that cause. Without the code I could not check that v6 never truncates or bounds its keys.
- maintenance-fit: 1 objection(s). The design is the one Robert Haas proposed in message #14, it fixes a regression from 8c49a484e8e, and it costs nothing measurable below the threshold. [P4] does not apply: this is the general fix it asked for. The single objection asks for a lasting test, not a different design.
- already-exists: 0 objection(s). No existing feature, setting or extension avoids this cost. The slowdown is inside choose_plan_name(), which every statement with subplans calls, so users cannot work around it and an extension cannot replace it. v6 reuses simplehash with a 32-name threshold, the pattern ec_derives_hash in equivclass.c already uses, so it adds no new mechanism. [P4] concerned a static cache shared by all equivalence classes and does not apply.
- demand: 0 objection(s). The demand case holds because this fixes a planning-time regression from v18, in code new in 19. Generated SQL with a few hundred UNION arms or sublinks that plans in milliseconds on 18 would plan 2x to 1000x slower after upgrade (message #5, measurements.txt). No field report is shown, but one independent developer hit it during other planner work (message #2). Robert Haas, who wrote the code, accepted fixing it and proposed the v6 design (message #14). The cost falls only on statements with 32 or more subplan names.

### Precedents retrieved

- P1: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P2: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P3: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P4: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - used in the review; reason summarised from its thread
- P5: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P6: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant; reason summarised from its thread
- P7: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant; reason summarised from its thread
- P8: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - judged not relevant; reason summarised from its thread
- P9: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P11: [Improve Append/MergeAppend EXPLAIN output when all subplans are pruned.](https://commitfest.postgresql.org/patch/2102/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P12: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P13: [Safer hash table initialization macro](https://commitfest.postgresql.org/patch/6276/) (cf, Withdrawn, 2026-03-09) - judged not relevant
- P14: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant
- P15: [psql: add tab completion for \df slash command suffixes](https://commitfest.postgresql.org/patch/2268/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P16: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P17: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2954/) (cf, Withdrawn, 2021-02-19) - judged not relevant
- P18: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P19: [More stable query plans via more predictable column statistics](https://commitfest.postgresql.org/patch/434/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P20: [Fix MINGW build on ARM64 on Windows](https://commitfest.postgresql.org/patch/5610/) (cf, Returned with feedback, 2025-04-07) - judged not relevant
- P21: [Revert "Improve accounting for memory used by shared hash tables"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aff1dc8dfa7584ddbc695e1aae0f767db66326f) (revert, Reverted, 2025-04-04) - judged not relevant
- P22: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2975/) (cf, Returned with feedback, 2021-03-27) - judged not relevant
- P23: [optimize several list functions with SIMD intrinsics](https://commitfest.postgresql.org/patch/4238/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P24: [CRC32C Parallel Computation Optimization on ARM](https://commitfest.postgresql.org/patch/4620/) (cf, Returned with feedback, 2025-04-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 83 s; cost reported by the CLI: $1.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:56:42.664Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
