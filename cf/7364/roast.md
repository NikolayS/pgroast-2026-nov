# Design pre-review: Costing for parallel scans with few/single row produced in the outer side

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7364](https://commitfest.postgresql.org/patch/7364/); [thread](https://www.postgresql.org/message-id/flat/CAEze2Wi%2BJp26mhBJvrMshKGYNTtPcxzd1BWoEW_7%2BeH6saj3gQ%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is stated and a committer who knows parallel costing treats it as real, so thin demand (demand-1) alone does not stop review. The design does: effective_workers has no defined invariant (maintenance-fit-1), and v2 still costs 5 outer rows below 4 at 4 workers (standards-compat-1). Message #6 also reports wrong values for partial aggregates and Parallel Append, which the patch does not target (demand-2). The list needs to agree on the invariant before code review makes sense.

Not checked: Without the code, the value of effective_workers for paths that set only parallel_workers and the planning time cost of the change could not be checked. Messages #4 to #6 were read only through the objections that cite them.

## Prior attempts

None found before 2026-09-28.

## Objections a reviewer is likely to raise

### 1. Every path constructor must now set effective_workers, and nothing checks it

A later patch adds a new partial path type, or a custom scan provider builds partial paths, and copies parallel_workers without deriving effective_workers. The planner then computes a wrong parallel divisor and wrong Gather row counts. Nobody sees this until a user reports a bad plan, because nothing errors and the regression suite tests few such shapes.

v1 touched four files and v2 touched ten, because the value has to be propagated through joins, upper rels, set operations and Append. Message #4 and message #6 found wrong values in partial aggregates, partial DISTINCT and Append, each a place where propagation was wrong without any test failing. Message #6 notes the field has no defined invariant, so a future author has no rule to follow when adding a node type.

**Question:** What written invariant for effective_workers lets a future path constructor or extension set it correctly, and what would fail if it set it wrong?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#4](https://www.postgresql.org/message-id/CABXr29FUZJsb1xHbzY_5%3DC2nuvbCUXWCNmBKzZXTgjkFqzdKAQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Plans for existing parallel queries change, with cost falling as estimates rise

After upgrade, autoanalyze moves a join's outer estimate between 4 and 5 rows at 4 workers. The cheaper plan flips each time, because v2 costs R=5 below R=4. The DBA sees unstable plans with no setting change and no switch to get the old costing back.

Message #6 shows the R=4 to R=5 cost drop is still present in v2. Message #5 attributes the drop to the existing leader heuristic. Message #6 counters that effective_workers makes that heuristic depend on cardinality at a fixed worker count, so the effect is new. Message #3 also accepts that queries which underestimate below the worker count can plan worse. The proposal gives users no way to opt out or bound that change.

**Question:** Can the cost be made non-decreasing in estimated rows, and what do users with underestimated queries do if their parallel plans disappear after upgrade?

<sub>fixable, level 2, planner behaviour change; messages [#3](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Partial paths built by extensions get no defined effective_workers

A custom scan provider or FDW builds a partial path and sets parallel_workers, as it does today, but has no reason to set the new field. In messages #4 and #6, effective_workers = 0 gives a divisor of 1, so such a path would be costed with no worker discount and lose to core paths. Users would see only a plan change after a major upgrade.

v2 adds Path->effective_workers and makes get_parallel_divisor() depend on it, so out-of-core code that creates partial paths now has a new duty. The proposal does not say what value a path gets when its creator leaves the field unset, and nothing fails at build time or run time when it is missing. v2 also changes create_append_path and other signatures in pathnode.h and paths.h. Those changes break the build loudly, but a missing field assignment does not.

**Question:** What effective_workers does a partial path get when an extension sets only parallel_workers, and how would such an extension learn it must change?

<sub>fixable, level 2, ABI / extension API; messages [#4](https://www.postgresql.org/message-id/CABXr29FUZJsb1xHbzY_5%3DC2nuvbCUXWCNmBKzZXTgjkFqzdKAQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Gain is narrow while plan and API changes reach everyone

A user with a one-group partial aggregate or a Parallel Append over low-row partitions gets changed Gather row estimates and possibly a worse plan after upgrade. Extensions that call create_append_path stop compiling, though they never had the single-row outer case.

The benefit applies only when the estimated outer rows fall below the worker count (message #3). v2 changes create_append_path and other signatures, and Haibo Yan shows wrong estimates for partial aggregates and Append, queries the patch does not target (message #6). So the cost falls on all parallel plans and on extensions, while the gain stays with a small set of queries.

**Question:** Can the change be limited to the join paths above a near-single-row parallel scan, so partial aggregates, Append and extension APIs see no change?

<sub>fixable, level 1b, desirability; messages [#3](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29Gt%3Dgv1GiCNh3qEYPmuUgtCnDNKydmd4YHBafPT2y9dUw%40mail.gmail.com); `demand-2`</sub>

### 5. Demand rests on one benchmark and one older report

The project adds a new Path field that every future path type must set correctly. Few production users ever hit the single-row outer case it targets, so later planner patches carry the cost of keeping it right for little real gain.

The trigger is a TPCC benchmark at one vendor (message #1). Robert Haas searched and found one clear matching report; the other two cases he found fall outside the patch (messages #2, #3). He said the bigger Path is hard to justify "as things are" (message #2). The int16 change in v2 removes the size cost, but the semantic and maintenance cost stays.

**Question:** Which production workloads, besides the TPCC benchmark and the one older list report, get a parallel seq scan over a cheaper PK lookup because of this discount?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAEze2Wi%2BJp26mhBJvrMshKGYNTtPcxzd1BWoEW_7%2BeH6saj3gQ%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CA%2BTgmobUaa0OAkUBOy0E-BxkG1k%2B0FZacFzp93_sPJPDQgx3Tw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAEze2Wj8jMehea76X5F1a4eq8aQs-i1_%2B_HGcD-mcTDwfzT6NA%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAEze2WiCGjYMy8OOsSBnyHSia3zK8BBD%3D0vycwZJgCE6Co7Uag%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What invariant does effective_workers satisfy: useful workers, leader slots, or worker occupancy?
2. Can the cost be made non-decreasing in estimated outer rows at a fixed worker count, so that 5 rows never cost less than 4 at 4 workers?
3. How should effective_workers be computed for partial aggregates, partial DISTINCT and Parallel Append so that the Gather row estimates reported in message #6 stay correct?
4. What effective_workers does a partial path get when an extension sets only parallel_workers, and what fails if the value is wrong?
5. Which workloads, besides the TPCC benchmark and the one older report found in message #2, pick a parallel seq scan over a cheaper PK lookup because of this discount?

## Touch points

- **ABI / extension API:** v2 adds Path->effective_workers, narrows the worker fields to int16 and changes create_append_path and other signatures in pathnode.h and paths.h.
- **planner behaviour change:** The parallel divisor now depends on estimated rows, so costs and plans of existing parallel queries change.
- **backpatchability:** The entry is tagged Bugfix and the problem was seen on pg18, but the Path layout and signature changes cannot go into a stable branch.
- **maintenance burden ("who maintains this in 10 years"):** Every partial path constructor, in core and in extensions, must set effective_workers correctly, and nothing fails when it does not.
- **performance on the hot path:** The patch changes the Path struct and adds work per path, and no planning time or memory measurement is shown.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Demand rests on one benchmark and one older report

## Who would commit it, and when

- Owner: qualified committer engaged. Robert Haas takes part in the thread (message #2) and reviews the approach, but he says he is not sure what is best. The owner must know parallel query costing and partial path generation.
- Priority: needs design rethink; audience niche; review cost medium. The gain applies only when estimated outer rows fall below the worker count, and message #2 found one clear earlier report. v2 is about 42 KiB across ten optimizer files and still gives wrong values for Append and partial aggregates, so the meaning of the field must be settled first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
