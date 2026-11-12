# Design pre-review: Expand OR clauses in joins to UNION ALL paths

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7405](https://commitfest.postgresql.org/patch/7405/); [thread](https://www.postgresql.org/message-id/flat/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Demand is shown and the goal has support in messages #2 and #14, so levels 1a and 1b pass. Level 2 fails on open design points: prior-attempts-1 (no written correctness argument, the gap that stopped P4), standards-compat-1 (a query that works on master fails with the patch, deferred in message #10), and maintenance-fit-1 with standards-compat-2 (planning cost and the hard cutoff). The list needs answers to these before code review is useful.

Not checked: The code was not read, so the correctness of the IS NOT TRUE exclusion, the outer join reduction per arm, and the v6 guards could not be checked. Only 3 of 14 messages were shown in full; later messages are known through the brief and the objections.

## Prior attempts

- [Convert join OR clauses into UNION queries](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com) (same idea; Returned with feedback, 2018-10-02). "FWIW, my problem with this patch is that I remain unconvinced of the basic correctness of the transform (specifically the unique-ification approach)." ([message](https://www.postgresql.org/message-id/7872.1538443930%40sss.pgh.pa.us)) Different now: Duplicate rows are avoided by adding IS NOT TRUE on earlier arms, so no CTID sort/unique step is needed. Non-table RTEs are allowed, and the arms are wrapped as subquery scans under an UPPERREL_SETOP Append.
- [Rewrite undirected edge patterns in GRAPH_TABLE using UNION ALL](https://www.postgresql.org/message-id/flat/c172f9dd5704b5d32c1f573f6ac33566@data-bene.io) (similar idea; Returned with feedback, 2026-09-10). "If OR transformation is implemented in the query optimizer, the implementation in graph query is useless and, more seriously, misleading if the correct statistics were to favour retaining OR." ([message](https://www.postgresql.org/message-id/CAExHW5tbzsgykWN3j2qtny7HpXxAWzPDY_yq_Wzcub-ZLyHXcA%40mail.gmail.com)) Different now: The same author moved the OR-to-UNION ALL rewrite from the GRAPH_TABLE rewriter into the planner and made it cost based. That is what the earlier objection asked for.

## Objections a reviewer is likely to raise

### 1. No written correctness proof, the gap that stopped the 2017 patch

A user runs a query with an OR over a left join's nullable side and arms that can be NULL. One branch reduces the outer join or excludes rows differently from the original filter. The query silently returns extra or missing rows, and nobody notices until results are compared by hand.

The precedent's author stopped work because he was not convinced the transform was correct and wanted a proof. This proposal replaces CTID unique-ification with IS NOT TRUE exclusion, which removes that specific step. Message #1 points to the old thread's last message for the proof but does not restate it, and it adds per-branch outer join reduction and more RTE kinds (message #10). The thread contains no argument covering those cases.

Precedent: [Convert join OR clauses into UNION queries](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com) (Returned with feedback, 2018-10-02)

**Question:** Where is a written argument that per-branch outer join reduction and IS NOT TRUE exclusion keep rows and multiplicity identical for every RTE kind the v6 guards allow?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#10](https://www.postgresql.org/message-id/f84fdea1-a0b9-4162-97b1-435992ff3b02%40data-bene.io); `prior-attempts-1`</sub>

### 2. OR expansion makes queries that work today raise errors after upgrade

An application runs `where i.kind = 'txt' or (a.val::int = 7 and a.tag = 7)` without error on the current release. After upgrade, the planner picks the Append, the second arm evaluates the cast for 'txt' rows, and the query fails with an invalid input error. The user finds out in production when a cost change flips the plan.

Message #9 shows such a query that works on master and fails with the patch. Message #10 defers it as "noted to be discussed later", and no version says how it will be handled. The docs do not guarantee evaluation order, but in practice the executor evaluates OR arms left to right and stops early, and existing code depends on that. Because the choice is cost based, the same query can work on one day and fail on the next, so a committer would likely ask for a policy before review continues.

**Question:** How will the patch avoid new errors in arms or negated predicates that the current executor never evaluates, for example by excluding arms with expressions that can fail?

<sub>fixable, level 2, design; messages [#9](https://www.postgresql.org/message-id/CAN4CZFOPWhh11CD8koPZ9NndemDY2og5qZeMMxmNvvUg6L5uMg%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/f84fdea1-a0b9-4162-97b1-435992ff3b02%40data-bene.io); `standards-compat-1`</sub>

### 3. Extra planner runs for every eligible query, with a hard arm cutoff

An OLTP application runs many short joins with an OR over two tables. After upgrade, each query runs one extra query_planner() per arm, and planning time roughly doubles even when the plain join wins. The user sees this as higher latency, and the only remedy is a server-wide GUC.

Message #1 reports planning going from 0.26 ms to 0.52 ms and from 1.6 ms to 3.0 ms, and that cost is paid whether or not the Append is picked. Message #5 showed exponential planning time with nesting, and subqueries are now skipped to avoid it. Message #14 asks for graceful degradation in place of or_expansion_limit, and no answer is given. [P4] also recorded a case with planning 170 times slower.

Precedent: [Convert join OR clauses into UNION queries](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com) (Returned with feedback, 2018-10-02)

**Question:** What planning time and memory do v6 queries show near or_expansion_limit when the Append loses, and how is that cost bounded without a hard cutoff?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#5](https://www.postgresql.org/message-id/CAN4CZFOFjDo-ACBCNuriV290%2B2kFDuUisMy10KQx71GB_t16Fw%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CA%2BTgmoY0vDEwDc7B9vbum_RXbeFh7OkhuyCJ3kAbkXdht49auw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. or_expansion_limit is a hard cutoff that hides a design decision

A report query has eight OR arms and gets the fast Append plan. A developer adds a ninth arm, expansion is silently skipped, and runtime goes back to the nested loop, which can be hundreds of times slower. The user sees only the slowdown and has no hint that a GUC caused it.

Message #1 picks the default of 8 as "a nice guess" and also calls the GUC an off switch for before/after comparisons, which is a development aid. Message #14 asks for graceful degradation in place of a sharp edge, and v6 does not answer. Message #5 shows planning cost growing exponentially with nesting, so the GUC limits only the number of arms and does not bound the cost it was added to control.

Precedent: [Convert join OR clauses into UNION queries](https://www.postgresql.org/message-id/flat/7f70bd5a-5d16-e05c-f0b4-2fdfc8873489@BlueTreble.com) (Returned with feedback, 2018-10-02)

**Question:** Can the cost of expansion be bounded by a planner-internal measure so that adding one OR arm does not abruptly turn off the plan?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/d606f08e-3a56-409f-8591-ccdbb3379b0e%40data-bene.io), [#5](https://www.postgresql.org/message-id/CAN4CZFOFjDo-ACBCNuriV290%2B2kFDuUisMy10KQx71GB_t16Fw%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CA%2BTgmoY0vDEwDc7B9vbum_RXbeFh7OkhuyCJ3kAbkXdht49auw%40mail.gmail.com); `standards-compat-2`</sub>

### 5. Cloned per-arm planner state must track every future PlannerInfo field

A later patch adds state that query_planner() fills in on the PlannerInfo and that setrefs or createplan reads from the top-level root. Nobody updates planorexpand.c, and the per-arm copy loses it. A user's query fails at execution with an internal error long after release.

Message #6 describes this happening already: minmax_aggs lived in the arm's subroot, setrefs looked in the top root, and the result was "Aggref found in non-Agg plan node". Message #12 found a second case, with processed_tlist out of sync with the arm's output. Each was fixed with a guard or a resync, and nothing makes the next missed field fail at plan time.

**Question:** What mechanism, beyond case-by-case guards, makes a new PlannerInfo field that the per-arm copy does not handle fail loudly in tests?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/f5ac4d86-76a2-4a5d-bf5d-3be211463d84%40data-bene.io), [#12](https://www.postgresql.org/message-id/0605f7a2-9b26-4125-bbf0-40ecfc007ce1%40data-bene.io); `maintenance-fit-3`</sub>

## Questions for the author

1. Where is a written argument that IS NOT TRUE exclusion and per-branch outer join reduction keep rows and multiplicity identical for every RTE kind the v6 guards allow?
2. How will the patch avoid new errors from arms or negated predicates that the executor does not evaluate today, such as the failing cast in message #9?
3. What planning time and memory does v6 show for queries with 6 to 8 arms over many relations when the Append loses?
4. Can a planner-internal measure bound the expansion cost, so that adding a ninth OR arm does not silently turn the plan off?
5. Will pg_plan_advice be able to request or forbid OR expansion and control each arm, or is expansion kept out of its scope?

## Touch points

- **ABI / extension API:** Extensions that walk plan trees, such as pg_plan_advice, see an UPPERREL_SETOP Append over synthetic subquery scans in a join rel's path list.
- **new GUC:** The patch adds or_expansion_limit (default 8, 0 disables).
- **planner behaviour change:** The planner adds an Append of per-arm plans as a competing path for queries with a cross-relation OR.
- **security / privileges:** The patch skips expansion when security quals are present, so RLS and security barrier views are a stated exclusion.
- **backpatchability:** A query that runs on master can raise an error after upgrade when the planner picks the Append (message #9).
- **maintenance burden ("who maintains this in 10 years"):** planorexpand.c clones planner state per arm and must stay in sync with PlannerInfo and with pg_plan_advice.
- **performance on the hot path:** Each OR arm costs one extra query_planner() run, paid even when the Append is not chosen.

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas commented in message #14 on test status, planning cost and pg_plan_advice, but has not offered to commit. The owner needs to know join planning, outer join reduction and set operation paths.
- Priority: fix design first; audience a sizeable minority; review cost high. Message #2 says such queries are frequent and the measured gains are large, so the goal is worth reviewer time. The v6 patch is about 100 KiB in the core planner, and correctness, error behaviour and planning cost are still open, so line-by-line review is premature.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
