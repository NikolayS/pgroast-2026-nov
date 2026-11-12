# Design pre-review: HashAggregate loses worker spill stats on rescan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7479](https://commitfest.postgresql.org/patch/7479/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWW5evhmWDWAgknx%3DGXNMJG%3Dk4-T%2Bd435pZpg8Lqxc%3DkFQ%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The only surviving objection, standards-compat-1, is a fixable scoping question at level 2 and does not stop review. The problem is stated with a reproducer, the fix copies how non-parallel HashAggregate counters behave across rescans, and the cost falls only on EXPLAIN ANALYZE of rescanned parallel plans. A committer who knows executor instrumentation should confirm the merge rule and the stable-branch target.

Not checked: Without the code, the review could not check that the initial batch counts once in every rescan and worker-reuse case. It also could not check that the patch changes no shared struct, which matters for back branches.

## Prior attempts

- [Revert "Fix accumulation of parallel worker instrumentation."](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com) (similar idea; Reverted, 2017-12-13). "Per further discussion, that doesn't seem to be the best possible fix." Different now: P6 changed how generic parallel worker instrumentation accumulates across Gather rescans. This patch changes only the HashAggregate-specific shared stats slot in ExecEndAgg().

## Objections a reviewer is likely to raise

### 1. Rescan semantics fixed for HashAggregate only, other worker-instrumented nodes left unchanged

A user runs EXPLAIN ANALYZE on a plan where Gather is rescanned and the workers run both a HashAggregate and a Sort or Memoize. The HashAggregate worker lines would then cover all executions, while the sibling node's worker lines may still cover only the last one. The user compares the two and reaches wrong conclusions about which node spilled.

The patch touches only nodeAgg.c. The proposal does not say whether Sort, Memoize, Hash or IncrementalSort also write per-worker slots that a Gather rescan reuses. If they do, one EXPLAIN output would mix two meanings for worker stats, and with a stable-branch target that mix would ship in minor releases. Matching the non-parallel HashAggregate counters is a sound rule, but a committer would likely ask that it apply to every node with shared worker instrumentation, or that the proposal say why only HashAggregate needs it.

**Question:** Do Sort, Memoize, Hash and IncrementalSort lose per-worker stats on Gather rescan, and if so, why fix only HashAggregate?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAJTYsWW5evhmWDWAgknx%3DGXNMJG%3Dk4-T%2Bd435pZpg8Lqxc%3DkFQ%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJTYsWW7DuqJao_sXmXrYh8TpTYQRB6LosN4_XMRjhMY0cM0iw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Do Sort, Memoize, Hash and IncrementalSort lose per-worker stats on a Gather rescan, and if so, why does the patch fix only HashAggregate?
2. Which stable branches does the fix target, and is changed EXPLAIN ANALYZE worker output acceptable in a minor release?
3. Can the reproducer from message #1 become a regression test with stable output for parallel HashAggregate rescans?
4. How should a reader relate the combined memory, disk and batch values to the per-loop averages that EXPLAIN shows for the same node?

## Touch points

- **backpatchability:** The CommitFest entry targets stable branches, and the fix changes EXPLAIN ANALYZE worker output in minor releases.
- **maintenance burden ("who maintains this in 10 years"):** ExecEndAgg() gets a HashAggregate-specific merge rule for the shared worker slot that later changes must keep correct.

## Who would commit it, and when

- Owner: silent. Someone who knows parallel executor instrumentation and the HashAggregate spill code. No committer has spoken in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch is a 1.8 KB change to nodeAgg.c that fixes wrong EXPLAIN ANALYZE output for rescanned Gather over HashAggregate. Only users who read such plans benefit, and the remaining point is a scoping question.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
