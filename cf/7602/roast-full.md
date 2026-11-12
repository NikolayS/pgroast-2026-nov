# Design pre-review: Propagate LIMIT bound through the outer side of left join

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7602](https://commitfest.postgresql.org/patch/7602/) (Needs review); authors: Andrei Lepikhov (lepikhov)
- Thread: [Propagate LIMIT bound through the outer side of left join](https://www.postgresql.org/message-id/flat/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), started 2026-10-29
- Messages read: 2 of 2
- Patch prose read: v3-0001-Propagate-LIMIT-bound-through-the-outer-side-of-l.patch, v2-0001-Propagate-LIMIT-bound-through-the-outer-side-of-l.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No surviving objection is fatal, and neither would stop a review at its level. maintenance-fit-1 is the decisive one: the bound is safe only while NestLoop and MergeJoin never fetch more outer tuples than they return, and the prose names no guard for that. demand-1 asks for evidence beyond one generated-table example. Both are fixable, but the thread has only the author's two messages and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm which quals block the bound, how rescans and parallel or Memoize nodes behave, or whether the join nodes carry a comment or Assert for the fetch-order property.

## The proposal, restated

**Problem.** ExecSetTupleBound() passes a LIMIT bound down to a Sort node for a bounded top-N sort, but stops at any join. So a query like `select ... from (select ... order by x) s left join b ... limit 10` fully sorts the outer input. The author targets queries with LIMIT over a LEFT or FULL join with a sorted outer side.

**Proposed design.** At executor start, ExecSetTupleBound() passes the bound unchanged to the outer side of a LEFT or FULL NestLoop or MergeJoin. Such a join emits at least one tuple per outer tuple before fetching the next one. The patch applies this only when the join has no qual on its output, and v3 skips HashJoin because a multi-batch hash join can fetch more outer tuples than it returns.

**What the author claims.**
- The example query runs in about 32.6 ms patched against 63.8 ms on master after warm-up.
- The feature is independent of the earlier Top-N sort propagation thread and has its own benefit.
- Merge join is the most likely beneficiary, so most new regression tests force a merge join.
- Bounded sort can lose to quicksort when the query needs most input tuples, but the existing 'tuples > 2 × limit' costing already accepts this trade-off.
- A LEFT or FULL NestLoop or MergeJoin with no output qual never asks its outer side for more tuples than it returns.

**Thread state.** Only the author has posted, with v2 and then v3, which drops HashJoin. No reviewer has replied yet.

## 2. Prior attempts

### Pass down Limit bounds [take-2] (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/866/](https://commitfest.postgresql.org/patch/866/)
- **Thread:** [https://www.postgresql.org/message-id/flat/9A28C8860F777E439AA12E8AEA7694F8012505BC@BPXM15GP.gisp.nec.co.jp](https://www.postgresql.org/message-id/flat/9A28C8860F777E439AA12E8AEA7694F8012505BC@BPXM15GP.gisp.nec.co.jp), [https://www.postgresql.org/message-id/flat/CAM2+6=VBkoPE54U_jGr8iSuR7Xe+BnwesiUK1tye3hckcy_AyQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAM2+6=VBkoPE54U_jGr8iSuR7Xe+BnwesiUK1tye3hckcy_AyQ@mail.gmail.com)
- **Date:** decided 2016-12-02; started 2016-10-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Jeevan Chalke, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers asked for changes. ps_numTuples should be a 64-bit type, and 0 should mean "unknown" so most nodes need not set the field. Robert Haas also asked the patch to keep planning-time use of constant LIMIT values separate from the execution-time mechanism. The author said he would revise the patch, but no revised version appears in the visible messages before the entry was returned with feedback.
- **Reason (quote):** "I think ps_numTuples should be int64." ([message](https://www.postgresql.org/message-id/CAM2%2B6%3DWb7a41%3DwJbHMfDqkK%2Bqh94RPNTKM99hGNFhQ9YUATu6A%40mail.gmail.com))
- **What is different this time:** P6 added a new PlanState field to carry a Limit hint to child nodes and to postgres_fdw. This proposal adds no field or API; it extends the existing ExecSetTupleBound() through the outer side of LEFT and FULL NestLoop and MergeJoin.


## 3. Objections a reviewer is likely to raise

### 1. Correctness rests on an unenforced fetch-order property of join nodes

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch changes NestLoop or MergeJoin so that it reads outer tuples ahead, for example batched outer fetches or new prefetching. The bounded Sort below then returns only N tuples while the join needs more, and the query silently returns fewer rows. No test or assertion catches this until a user reports wrong results.

The commit message bases the whole optimization on one property: a LEFT/FULL NestLoop or MergeJoin never fetches more outer tuples than it returns. A bounded Sort cuts off its output at the bound, so a violation gives wrong results, not just lost speed. v3 already had to drop HashJoin because multi-batch mode breaks this property (message #2), which shows how easily join code can break it. The prose does not say whether the patch adds a comment or an Assert in the join nodes so that a future change there fails loudly.

Thread: [#1 (Andrei Lepikhov, 2026-10-29)](https://www.postgresql.org/message-id/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), [#2 (Andrei Lepikhov, 2026-11-03)](https://www.postgresql.org/message-id/1c0860b6-a1f8-40ed-867b-0ddad458d095%40gmail.com)

**Question:** How will a future change to NestLoop or MergeJoin outer fetching be detected as breaking this bound, through an Assert, a runtime check, or a test that fails?

Counter-check: held. ExecSetTupleBound() already relies on unasserted properties of other nodes, but a join's fetch order is subtler, and v3 dropped HashJoin for breaking it. A bounded Sort truncates its output, so a later violation gives wrong results, and the prose names no guard.

### 2. Demand rests on one synthetic query shape

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer takes on a new executor contract: a LEFT or FULL NestLoop or MergeJoin must never fetch more outer tuples than it returns. The material shows no production query that gains from it. If the contract later breaks for a join that gains new behaviour, users get wrong LIMIT results, and the benefit was never shown outside a test.

The only evidence is the generated-table example in message #1, about 64 ms against 33 ms. That example puts ORDER BY in a subquery and LIMIT outside it with no top-level ORDER BY, which is not how users usually write top-N queries. No field report, -general or -bugs thread, or second user appears, and no reviewer has replied (messages #1 and #2). The author also notes that bounded sort can be slower than quicksort, and no case measures that slowdown.

Thread: [#1 (Andrei Lepikhov, 2026-10-29)](https://www.postgresql.org/message-id/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), [#2 (Andrei Lepikhov, 2026-11-03)](https://www.postgresql.org/message-id/1c0860b6-a1f8-40ed-867b-0ddad458d095%40gmail.com)

**Question:** Which real query shapes, such as a top-level order by over a merge left join, gain from this, and how often do they appear in practice?

Counter-check: held. The thread shows one generated-table example, no slowdown measurement and no second voice. Its wrong-results risk repeats maintenance-fit-1 and adds no weight here; the cost is otherwise small, so evidence of common plan shapes that gain would answer it.

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
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | ExecSetTupleBound() now depends on NestLoop and MergeJoin never fetching more outer tuples than they return. |
| performance on the hot path | **yes** | A Sort below a LEFT or FULL join under LIMIT becomes a bounded sort, which the author says can lose to quicksort. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Demand rests on one synthetic query shape |

## 5. Questions the author must answer before anyone reviews code

1. How will a future change to outer-tuple fetching in NestLoop or MergeJoin be caught: by an Assert, a runtime check, or a regression test that fails?
2. Which query shapes that users write, such as a top-level order by with limit over a merge left join, gain from this patch?
3. What is the measured slowdown against master when the bounded sort is chosen but the query needs most of the outer tuples?
4. What happens to the bound when the join is rescanned, or when Gather, Gather Merge or Memoize sits between the join and the Sort?
5. Which quals block the bound: only quals on the join output, or also join quals, and are semi and anti joins excluded?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the executor's join nodes and tuplesort's bounded mode; none has posted in the thread.
- **Liability:** The owner signs for wrong LIMIT results if NestLoop or MergeJoin ever starts to read outer tuples ahead, for as long as this propagation exists.
- **Author follow-up:** The author posted v3 five days after v2 and removed HashJoin after finding the multi-batch problem without a reviewer's prompt.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch touches one executor file plus regression tests and adds no user-visible surface, so review is cheap. The thread shows one generated-table example, and the fetch-order property of the join nodes needs care in later join work.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent gives a rejection reason that still applies. P6 was returned over its field type, a sentinel value and the split between planning and execution, and an executor-only extension of ExecSetTupleBound() raises none of these. Message #1 links an earlier Top-N sort propagation thread and calls this a separate feature; the visible material does not say how that thread ended.
- standards-compat: 0 objection(s). The patch only changes how the executor passes an existing LIMIT bound, and it adds no syntax, keyword, GUC, catalog object, pg_dump state or psql change. Visible effects are EXPLAIN ANALYZE showing top-N heapsort instead of a full sort, and possibly a different order for tied sort keys under a LIMIT without a top-level ORDER BY, which is already unspecified. Bounded sorts below Limit work this way today.
- replication-ondisk: 0 objection(s). The patch changes only executor-time propagation in ExecSetTupleBound() plus regression tests. It adds no on-disk format, WAL record, catalog, wire protocol or replication state. PlanState fields are reused, so no extension ABI change is visible from the material; the code would need to confirm this.
- security: 0 objection(s). The patch adds no object, privilege, setting or monitoring output. It only lets fewer outer tuples be read and sorted under a LIMIT, so no user can see or run more than on master. Security barrier and RLS quals on the join output block the bound, since the patch requires no qual there. Whether quals below the join can still leak was not checked without the code.
- maintenance-fit: 1 objection(s). The change is small, sits in execProcnode.c, and adds no cost for queries without LIMIT. No committer has engaged yet. That affects committability and is not counted against the design.
- already-exists: 0 objection(s). No existing feature gives this result automatically. A user can move the LIMIT into the outer subquery by hand, but that needs a rewrite, which ORM-generated queries may not allow. The patch also extends the existing ExecSetTupleBound() mechanism rather than adding a second one, and an extension cannot change this executor path.
- demand: 1 objection(s). Cost to non-users looks small, since the change is limited to execProcnode.c and does not touch the planner. The only gap is evidence of demand beyond one synthetic example.

### Precedents retrieved

- P1: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant; reason summarised from its thread
- P2: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P3: [Table Partition + Join Pushdown](https://commitfest.postgresql.org/patch/325/) (cf, Rejected, 2016-01-25) - judged not relevant; reason summarised from its thread
- P4: [Range Merge Join](https://commitfest.postgresql.org/patch/3160/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P5: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant; reason summarised from its thread
- P6: [Pass down Limit bounds \[take-2\]](https://commitfest.postgresql.org/patch/866/) (cf, Returned with feedback, 2016-12-02) - used in the review; reason summarised from its thread
- P7: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P8: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant; reason summarised from its thread
- P9: [Range Merge Join](https://commitfest.postgresql.org/patch/1106/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P10: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/20/) (cf, Rejected, 2015-01-16) - judged not relevant
- P11: [Range Merge Join](https://commitfest.postgresql.org/patch/1449/) (cf, Returned with feedback, 2018-03-28) - judged not relevant
- P12: [Add a greedy join search algorithm to handle large join problems](https://commitfest.postgresql.org/patch/6486/) (cf, Withdrawn, 2026-02-14) - judged not relevant; reason summarised from its thread
- P13: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant
- P14: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P15: [Increasing the default values of from_collapse_limit, join_collapse_limit, and/or geqo_threshold](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P18: [Implement predicate propagation for non-equivalence clauses.](https://commitfest.postgresql.org/patch/1807/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P19: [Push down aggregates below joins.](https://commitfest.postgresql.org/patch/1685/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [Remove LEFT JOINs in more cases](https://commitfest.postgresql.org/patch/1364/) (cf, Returned with feedback, 2018-09-28) - judged not relevant
- P21: [Automatic notification of top transaction IDs](https://commitfest.postgresql.org/patch/3232/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P22: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant
- P23: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant
- P24: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 149 s; cost reported by the CLI: $1.19
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 55 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:05:15.091Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
