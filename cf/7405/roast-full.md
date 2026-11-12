# Design pre-review: Expand OR clauses in joins to UNION ALL paths

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7405](https://commitfest.postgresql.org/patch/7405/) (Needs review); authors: Ayoub Kazar (kaz)
- Thread: [\[PROPOSAL\] Expand OR clauses in joins to UNION ALL paths](https://www.postgresql.org/message-id/flat/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), started 2026-10-07
- Messages read: 14 of 14
- Patch prose read: v6-0001-Expand-OR-clauses-in-joins-to-UNION-ALL-Append-paths.patch, 0001-Expand-OR-clauses-in-joins-to-UNION-ALL-Append-paths.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Demand is shown and the goal has support in messages #2 and #14, so levels 1a and 1b pass. Level 2 fails on open design points: prior-attempts-1 (no written correctness argument, the gap that stopped P4), standards-compat-1 (a query that works on master fails with the patch, deferred in message #10), and maintenance-fit-1 with standards-compat-2 (planning cost and the hard cutoff). The list needs answers to these before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the correctness of the IS NOT TRUE exclusion, the outer join reduction per arm, and the v6 guards could not be checked. Only 3 of 14 messages were shown in full; later messages are known through the brief and the objections.

## The proposal, restated

**Problem.** An OR that spans relations cannot be a hash or merge join clause, and BitmapOr only combines indexes of one relation. So such queries end up as nested loops with per-row inner scans or join filters. The author hit this with SQL/PGQ undirected edges and cites star-schema queries with an OR over two dimension tables.

**Proposed design.** For a top-level WHERE OR with arms C1..Cn, the planner plans one copy of the query per arm, with Ck plus "(Cj) IS NOT TRUE" for every earlier arm, so no deduplication is needed. In v6 each arm is a synthetic subquery scan, and the arms go into an Append on an UPPERREL_SETOP rel that is added to the join rel's path list. Normal costing decides whether the Append is used.

**User-visible surface.**
- New GUC or_expansion_limit (default 8, 0 disables) caps the number of OR arms considered.
- New plan shape: Append over per-arm subquery scans in place of a join, visible in EXPLAIN and to pg_plan_advice.
- Queries that rely on OR short-circuit evaluation can now raise errors, for example a cast that fails in an arm not evaluated on master.

**What the author claims.**
- The SQL/PGQ case on LSQB SF 0.1 went from 91.7 s to 1.6 s when written by hand as UNION ALL, and up to 4610x faster at larger scale factors.
- Star schema with a 20M row fact table: 1893 ms to 148 ms, with planning time 0.26 ms to 0.52 ms.
- OR join condition on two 2M row tables: 1520 ms to 435 ms, with about 300 times fewer buffer hits.
- Original plans are kept when arms are not selective (about 20% of rows each).
- Negated earlier arms give the same rows and multiplicity as the original query, which is simpler than Tom Lane's 2017 CTID-based deduplication.

**Thread state.** David Geier supports the idea and Zsolt Parragi reported test failures, an unhandled subplan case, exponential planning time with nested subqueries, and short-circuit errors, so the patch went through v6. Robert Haas said the work is about 1% done and asked for passing tests, fuzz testing, graceful handling of planning cost, and fewer arbitrary exclusions.

## 2. Prior attempts

### Convert join OR clauses into UNION queries (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/1001/](https://commitfest.postgresql.org/patch/1001/)
- **Thread:** [https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com)
- **Date:** decided 2018-10-02; started 2017-02-08
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Noah Misch (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane, the patch author, said he was still not convinced that the unique-ification step is correct, and he would not work on it further without a proof. Noah Misch's review, with assertion failures and a case where planning was 170 times slower and the chosen plan 4.5 times slower, got no reply for a month, so the entry was returned with feedback.
- **Reason (quote):** "FWIW, my problem with this patch is that I remain unconvinced of the basic correctness of the transform (specifically the unique-ification approach)." ([message](https://www.postgresql.org/message-id/7872.1538443930%40sss.pgh.pa.us))
- **What is different this time:** Duplicate rows are avoided by adding IS NOT TRUE on earlier arms, so no CTID sort/unique step is needed. Non-table RTEs are allowed, and the arms are wrapped as subquery scans under an UPPERREL_SETOP Append.

### Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/7034/](https://commitfest.postgresql.org/patch/7034/)
- **Thread:** [https://www.postgresql.org/message-id/flat/c172f9dd5704b5d32c1f573f6ac33566@data-bene.io](https://www.postgresql.org/message-id/flat/c172f9dd5704b5d32c1f573f6ac33566@data-bene.io)
- **Date:** decided 2026-09-10; started 2026-07-16
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Ashutosh Bapat (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Ashutosh Bapat argued that the OR-to-UNION transformation belongs in the planner, where it would help all join-with-OR queries and could be cost based. An unconditional rewrite in the graph query rewriter would be redundant once the planner does it, and misleading when statistics favour keeping the OR. The author agreed the work should move to the optimizer as a separate patch. SQL/PGQ was also reverted from master, leaving the patch nothing to apply against.
- **Reason (quote):** "If OR transformation is implemented in the query optimizer, the implementation in graph query is useless and, more seriously, misleading if the correct statistics were to favour retaining OR." ([message](https://www.postgresql.org/message-id/CAExHW5tbzsgykWN3j2qtny7HpXxAWzPDY_yq_Wzcub-ZLyHXcA%40mail.gmail.com))
- **What is different this time:** The same author moved the OR-to-UNION ALL rewrite from the GRAPH_TABLE rewriter into the planner and made it cost based. That is what the earlier objection asked for.


## 3. Objections a reviewer is likely to raise

### 1. No written correctness proof, the gap that stopped the 2017 patch

fixable · level 2, rubric item 7 · prior rejection, planner behaviour change · `prior-attempts-1`

A user runs a query with an OR over a left join's nullable side and arms that can be NULL. One branch reduces the outer join or excludes rows differently from the original filter. The query silently returns extra or missing rows, and nobody notices until results are compared by hand.

The precedent's author stopped work because he was not convinced the transform was correct and wanted a proof. This proposal replaces CTID unique-ification with IS NOT TRUE exclusion, which removes that specific step. Message #1 points to the old thread's last message for the proof but does not restate it, and it adds per-branch outer join reduction and more RTE kinds (message #10). The thread contains no argument covering those cases.

**Precedents.**
- [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com)

Thread: [#1 (Ayoub Kazar, 2026-10-07)](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#10 (Ayoub Kazar, 2026-10-27)](https://www.postgresql.org/message-id/f84fdea1-a0b9-4162-97b1-435992ff3b02%40data-bene.io)

**Question:** Where is a written argument that per-branch outer join reduction and IS NOT TRUE exclusion keep rows and multiplicity identical for every RTE kind the v6 guards allow?

Counter-check: held. Message #1 points to the old thread for the proof and does not restate it. No message argues correctness for per-branch outer join reduction or the RTE kinds added in message #10, so P4's recorded reason still applies.

### 2. OR expansion makes queries that work today raise errors after upgrade

fixable · level 2, rubric item 6 · design, planner behaviour change, backpatchability · `standards-compat-1`

An application runs `where i.kind = 'txt' or (a.val::int = 7 and a.tag = 7)` without error on the current release. After upgrade, the planner picks the Append, the second arm evaluates the cast for 'txt' rows, and the query fails with an invalid input error. The user finds out in production when a cost change flips the plan.

Message #9 shows such a query that works on master and fails with the patch. Message #10 defers it as "noted to be discussed later", and no version says how it will be handled. The docs do not guarantee evaluation order, but in practice the executor evaluates OR arms left to right and stops early, and existing code depends on that. Because the choice is cost based, the same query can work on one day and fail on the next, so a committer would likely ask for a policy before review continues.

Thread: [#9 (Zsolt Parragi, 2026-10-12)](https://www.postgresql.org/message-id/CAN4CZFOPWhh11CD8koPZ9NndemDY2og5qZeMMxmNvvUg6L5uMg%40mail.gmail.com), [#10 (Ayoub Kazar, 2026-10-27)](https://www.postgresql.org/message-id/f84fdea1-a0b9-4162-97b1-435992ff3b02%40data-bene.io)

**Question:** How will the patch avoid new errors in arms or negated predicates that the current executor never evaluates, for example by excluding arms with expressions that can fail?

Counter-check: held. Message #9 gives a query that works on master and fails with the patch, and message #10 defers it. The documentation does not guarantee evaluation order, which lowers the weight, but the thread still has no stated policy.

### 3. Extra planner runs for every eligible query, with a hard arm cutoff

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change, new GUC · `maintenance-fit-1`

An OLTP application runs many short joins with an OR over two tables. After upgrade, each query runs one extra query_planner() per arm, and planning time roughly doubles even when the plain join wins. The user sees this as higher latency, and the only remedy is a server-wide GUC.

Message #1 reports planning going from 0.26 ms to 0.52 ms and from 1.6 ms to 3.0 ms, and that cost is paid whether or not the Append is picked. Message #5 showed exponential planning time with nesting, and subqueries are now skipped to avoid it. Message #14 asks for graceful degradation in place of or_expansion_limit, and no answer is given. [P4] also recorded a case with planning 170 times slower.

**Precedents.**
- [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com)

Thread: [#1 (Ayoub Kazar, 2026-10-07)](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#5 (Zsolt Parragi, 2026-10-07)](https://www.postgresql.org/message-id/CAN4CZFOFjDo-ACBCNuriV290%2B2kFDuUisMy10KQx71GB_t16Fw%40mail.gmail.com), [#14 (Robert Haas, 2026-11-06)](https://www.postgresql.org/message-id/CA%2BTgmoY0vDEwDc7B9vbum_RXbeFh7OkhuyCJ3kAbkXdht49auw%40mail.gmail.com)

**Question:** What planning time and memory do v6 queries show near or_expansion_limit when the Append loses, and how is that cost bounded without a hard cutoff?

Counter-check: held. Message #1 shows planning time about doubling, and that cost is paid even when the Append loses. It falls only on queries with a cross-relation OR, but the thread has no v6 measurements near the limit.

### 4. or_expansion_limit is a hard cutoff that hides a design decision

fixable · level 2, rubric item 10 · new GUC, planner behaviour change · `standards-compat-2`

A report query has eight OR arms and gets the fast Append plan. A developer adds a ninth arm, expansion is silently skipped, and runtime goes back to the nested loop, which can be hundreds of times slower. The user sees only the slowdown and has no hint that a GUC caused it.

Message #1 picks the default of 8 as "a nice guess" and also calls the GUC an off switch for before/after comparisons, which is a development aid. Message #14 asks for graceful degradation in place of a sharp edge, and v6 does not answer. Message #5 shows planning cost growing exponentially with nesting, so the GUC limits only the number of arms and does not bound the cost it was added to control.

**Precedents.**
- [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (Returned with feedback, 2018-10-02); thread: [link](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com)

Thread: [#1 (Ayoub Kazar, 2026-10-07)](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#5 (Zsolt Parragi, 2026-10-07)](https://www.postgresql.org/message-id/CAN4CZFOFjDo-ACBCNuriV290%2B2kFDuUisMy10KQx71GB_t16Fw%40mail.gmail.com), [#14 (Robert Haas, 2026-11-06)](https://www.postgresql.org/message-id/CA%2BTgmoY0vDEwDc7B9vbum_RXbeFh7OkhuyCJ3kAbkXdht49auw%40mail.gmail.com)

**Question:** Can the cost of expansion be bounded by a planner-internal measure so that adding one OR arm does not abruptly turn off the plan?

Counter-check: held. Message #14 describes the same abrupt regression when one arm is added, and v6 keeps the cutoff. The nesting point is outdated, because v2 limits expansion to top-level queries, but the objection stands without it.

### 5. Cloned per-arm planner state must track every future PlannerInfo field

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-3`

A later patch adds state that query_planner() fills in on the PlannerInfo and that setrefs or createplan reads from the top-level root. Nobody updates planorexpand.c, and the per-arm copy loses it. A user's query fails at execution with an internal error long after release.

Message #6 describes this happening already: minmax_aggs lived in the arm's subroot, setrefs looked in the top root, and the result was "Aggref found in non-Agg plan node". Message #12 found a second case, with processed_tlist out of sync with the arm's output. Each was fixed with a guard or a resync, and nothing makes the next missed field fail at plan time.

Thread: [#6 (Ayoub Kazar, 2026-10-08)](https://www.postgresql.org/message-id/f5ac4d86-76a2-4a5d-bf5d-3be211463d84%40data-bene.io), [#12 (Ayoub Kazar, 2026-10-28)](https://www.postgresql.org/message-id/0605f7a2-9b26-4125-bbf0-40ecfc007ce1%40data-bene.io)

**Question:** What mechanism, beyond case-by-case guards, makes a new PlannerInfo field that the per-arm copy does not handle fail loudly in tests?

Counter-check: held. Messages #6 and #12 each report a case where per-arm planner state and the top-level root disagreed, and each was fixed with a guard or a resync. The thread shows no general mechanism that catches the next such field.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Extensions that walk plan trees, such as pg_plan_advice, see an UPPERREL_SETOP Append over synthetic subquery scans in a join rel's path list. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds or_expansion_limit (default 8, 0 disables). |
| planner behaviour change | **yes** | The planner adds an Append of per-arm plans as a competing path for queries with a cross-relation OR. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch skips expansion when security quals are present, so RLS and security barrier views are a stated exclusion. |
| backpatchability | **yes** | A query that runs on master can raise an error after upgrade when the planner picks the Append (message #9). |
| maintenance burden ("who maintains this in 10 years") | **yes** | planorexpand.c clones planner state per arm and must stay in sync with PlannerInfo and with pg_plan_advice. |
| performance on the hot path | **yes** | Each OR arm costs one extra query_planner() run, paid even when the Append is not chosen. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Where is a written argument that IS NOT TRUE exclusion and per-branch outer join reduction keep rows and multiplicity identical for every RTE kind the v6 guards allow?
2. How will the patch avoid new errors from arms or negated predicates that the executor does not evaluate today, such as the failing cast in message #9?
3. What planning time and memory does v6 show for queries with 6 to 8 arms over many relations when the Append loses?
4. Can a planner-internal measure bound the expansion cost, so that adding a ninth OR arm does not silently turn the plan off?
5. Will pg_plan_advice be able to request or forbid OR expansion and control each arm, or is expansion kept out of its scope?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas commented in message #14 on test status, planning cost and pg_plan_advice, but has not offered to commit. The owner needs to know join planning, outer join reduction and set operation paths.
- **Liability:** The owner signs for wrong-result risk in a new query transform, planning-time regressions, new runtime errors in existing queries, and keeping the per-arm planner copy and pg_plan_advice in sync with later planner changes, with no end date.
- **Author follow-up:** The author posted six versions in about five weeks in response to review, which suggests active follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Message #2 says such queries are frequent and the measured gains are large, so the goal is worth reviewer time. The v6 patch is about 100 KiB in the core planner, and correctness, error behaviour and planning cost are still open, so line-by-line review is premature.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) New Append plan shape is not covered by pg_plan_advice. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-2` Planning-time blowup recurs as in the earlier attempt. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same planning-cost point with the numbers from message #1 and asks for the same measurements near or_expansion_limit.
- `maintenance-fit-2` Set operation paths inside a join rel break assumptions elsewhere. Refuted (not specific): Duplicate of standards-compat-3: the only consumer it names is pg_plan_advice, and the rtekind mismatch it cites was a v5 error that v6 removed. It names no other component that would break.

### Review angles

- prior-attempts: 2 objection(s). [P7] gives no usable reason, so it is not used. The [P3] reason is resolved by this proposal.
- standards-compat: 3 objection(s). Three objections found within the standards and compatibility angle.
- replication-ondisk: 0 objection(s). The proposal changes only planning: it adds a GUC and a new Append path shape built inside one backend. It touches no on-disk format, WAL record, catalog, or wire protocol message, and the file list shows no struct change in the extension API, so this angle has no objection. Extensions that walk plan trees, such as pg_plan_advice, may see a new plan shape. Other panel members cover that.
- security: 0 objection(s). The proposal skips security quals, so RLS tables and security barrier views keep the normal plan, and arms only see columns the user may already read. Planning cost is a resource issue already reachable through join_collapse_limit, and the short-circuit errors involve only data the user can read.
- maintenance-fit: 3 objection(s). Objections listed above.
- already-exists: 0 objection(s). Hand-written UNION ALL with IS NOT TRUE guards is the only existing alternative, and generated SQL such as SQL/PGQ undirected edges and ORMs cannot use it. A cost-based choice also needs the planner; [P3] moved the work out of the rewriter for that reason, and messages #2 and #14 welcome the feature. BitmapOr and the OR-to-ANY conversion cover only single-relation ORs.
- demand: 0 objection(s). The material shows real demand: Tom Lane wrote a patch for the same transform in 2017 [P4], David Geier reports such queries occur frequently (message #2), and Robert Haas says he would like the feature (message #14). The SQL/PGQ undirected-edge case and the star-schema case are concrete workloads with large measured gains, so the open problems belong to design and correctness, not demand.

### Precedents retrieved

- P1: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P2: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P3: [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://commitfest.postgresql.org/patch/7034/) (cf, Returned with feedback, 2026-09-10) - used in the review; reason summarised from its thread
- P4: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - used in the review; reason summarised from its thread
- P5: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant; reason summarised from its thread
- P6: [Add table expansion hook](https://commitfest.postgresql.org/patch/2614/) (cf, Returned with feedback, 2021-09-13) - judged not relevant; reason summarised from its thread
- P7: [Union Replacement of OR logic](https://commitfest.postgresql.org/patch/4715/) (cf, Rejected, 2024-07-24) - used in the review; reason summarised from its thread
- P8: [doc: Add some OID/TOAST-related limitations to the limits appendix.](https://commitfest.postgresql.org/patch/4320/) (cf, Withdrawn, 2023-06-13) - judged not relevant; reason summarised from its thread
- P9: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant
- P10: [apply_scanjoin_target_to_paths and partitionwise join](https://commitfest.postgresql.org/patch/4929/) (cf, Returned with feedback, 2025-01-13) - judged not relevant
- P11: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P12: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P13: [pg_stat_lock: add blocker mode dimension](https://commitfest.postgresql.org/patch/6820/) (cf, Withdrawn, 2026-08-13) - judged not relevant
- P14: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P15: [rewriteGraphTable: Fix missing RTEs in FROM clause by setting inFromCl=true](https://commitfest.postgresql.org/patch/6602/) (cf, Withdrawn, 2026-09-10) - judged not relevant
- P16: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Hashjoin fall back to block nested hashloop join](https://commitfest.postgresql.org/patch/2588/) (cf, Withdrawn, 2020-11-10) - judged not relevant
- P18: [Parallel Append implementation](https://commitfest.postgresql.org/patch/929/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P19: [New GUC prepared_statement_limit to limit memory used by prepared statements](https://commitfest.postgresql.org/patch/2241/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P20: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P21: [\[PERF\] Improve Cardinality Estimation for Joins with GROUP BY Having Single Clause](https://commitfest.postgresql.org/patch/5422/) (cf, Withdrawn, 2025-04-06) - judged not relevant
- P22: [Make Append Cost aware of some run time partition prune case](https://commitfest.postgresql.org/patch/2829/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P23: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P24: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 145 s; cost reported by the CLI: $2.17
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 91 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:51:08.431Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
