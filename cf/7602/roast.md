# Design pre-review: Propagate LIMIT bound through the outer side of left join

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7602](https://commitfest.postgresql.org/patch/7602/); [thread](https://www.postgresql.org/message-id/flat/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No surviving objection is fatal, and neither would stop a review at its level. maintenance-fit-1 is the decisive one: the bound is safe only while NestLoop and MergeJoin never fetch more outer tuples than they return, and the prose names no guard for that. demand-1 asks for evidence beyond one generated-table example. Both are fixable, but the thread has only the author's two messages and the code was not read.

Not checked: Without the code, nobody can confirm which quals block the bound, how rescans and parallel or Memoize nodes behave, or whether the join nodes carry a comment or Assert for the fetch-order property.

## Prior attempts

- [Pass down Limit bounds \[take-2\]](https://www.postgresql.org/message-id/flat/9A28C8860F777E439AA12E8AEA7694F8012505BC@BPXM15GP.gisp.nec.co.jp) (similar idea; Returned with feedback, 2016-12-02). "I think ps_numTuples should be int64." ([message](https://www.postgresql.org/message-id/CAM2%2B6%3DWb7a41%3DwJbHMfDqkK%2Bqh94RPNTKM99hGNFhQ9YUATu6A%40mail.gmail.com)) Different now: P6 added a new PlanState field to carry a Limit hint to child nodes and to postgres_fdw. This proposal adds no field or API; it extends the existing ExecSetTupleBound() through the outer side of LEFT and FULL NestLoop and MergeJoin.

## Objections a reviewer is likely to raise

### 1. Correctness rests on an unenforced fetch-order property of join nodes

A later patch changes NestLoop or MergeJoin so that it reads outer tuples ahead, for example batched outer fetches or new prefetching. The bounded Sort below then returns only N tuples while the join needs more, and the query silently returns fewer rows. No test or assertion catches this until a user reports wrong results.

The commit message bases the whole optimization on one property: a LEFT/FULL NestLoop or MergeJoin never fetches more outer tuples than it returns. A bounded Sort cuts off its output at the bound, so a violation gives wrong results, not just lost speed. v3 already had to drop HashJoin because multi-batch mode breaks this property (message #2), which shows how easily join code can break it. The prose does not say whether the patch adds a comment or an Assert in the join nodes so that a future change there fails loudly.

**Question:** How will a future change to NestLoop or MergeJoin outer fetching be detected as breaking this bound, through an Assert, a runtime check, or a test that fails?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), [#2](https://www.postgresql.org/message-id/1c0860b6-a1f8-40ed-867b-0ddad458d095%40gmail.com); `maintenance-fit-1`</sub>

### 2. Demand rests on one synthetic query shape

A committer takes on a new executor contract: a LEFT or FULL NestLoop or MergeJoin must never fetch more outer tuples than it returns. The material shows no production query that gains from it. If the contract later breaks for a join that gains new behaviour, users get wrong LIMIT results, and the benefit was never shown outside a test.

The only evidence is the generated-table example in message #1, about 64 ms against 33 ms. That example puts ORDER BY in a subquery and LIMIT outside it with no top-level ORDER BY, which is not how users usually write top-N queries. No field report, -general or -bugs thread, or second user appears, and no reviewer has replied (messages #1 and #2). The author also notes that bounded sort can be slower than quicksort, and no case measures that slowdown.

**Question:** Which real query shapes, such as a top-level order by over a merge left join, gain from this, and how often do they appear in practice?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/e52eb9d1-e072-4b88-bfdf-7750d585fae3%40gmail.com), [#2](https://www.postgresql.org/message-id/1c0860b6-a1f8-40ed-867b-0ddad458d095%40gmail.com); `demand-1`</sub>

## Questions for the author

1. How will a future change to outer-tuple fetching in NestLoop or MergeJoin be caught: by an Assert, a runtime check, or a regression test that fails?
2. Which query shapes that users write, such as a top-level order by with limit over a merge left join, gain from this patch?
3. What is the measured slowdown against master when the bounded sort is chosen but the query needs most of the outer tuples?
4. What happens to the bound when the join is rescanned, or when Gather, Gather Merge or Memoize sits between the join and the Sort?
5. Which quals block the bound: only quals on the join output, or also join quals, and are semi and anti joins excluded?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** ExecSetTupleBound() now depends on NestLoop and MergeJoin never fetching more outer tuples than they return.
- **performance on the hot path:** A Sort below a LEFT or FULL join under LIMIT becomes a bounded sort, which the author says can lose to quicksort.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Demand rests on one synthetic query shape

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the executor's join nodes and tuplesort's bounded mode; none has posted in the thread.
- Priority: proceed; audience unclear; review cost low. The patch touches one executor file plus regression tests and adds no user-visible surface, so review is cheap. The thread shows one generated-table example, and the fetch-order property of the join nodes needs care in later join work.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
