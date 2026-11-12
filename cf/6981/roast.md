# Design pre-review: Lazy LEFT NestLoop JOIN: gate rescanning of the inner side by outer-side-only join clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6981](https://commitfest.postgresql.org/patch/6981/); [thread](https://www.postgresql.org/message-id/flat/CAKE%3DrqQ-LHuh2eVsKC7ihkRJoCBZafSR72o3Xk4Xb%3DLcQMQfsA%40mail.gmail.com), 23 of 23 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has shown demand, and the gating Result design follows what Tom Lane and Andres Freund suggested. Level 2 fails on open design points. prior-attempts-1: message #21 measures about 1500 ms against about 470 ms on master, and the planner makes no cost-based choice. standards-compat-1: a left join that returns rows on HEAD can raise division by zero, and the thread answers only with a link to a blog post. No reviewer has commented on opportunistic gating versus a cost-based path.

Not checked: The v4 code was not read. I could not check which clauses v4 gates (error-raising, stable or non-leakproof), whether semi joins are affected, or how keep_inner_rewind interacts with Materialize rescans.

## Prior attempts

- [Improve the performance of nested loop join in the case of partitioned inner table](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com) (related; Returned with feedback, 2023-08-01). "In short, I think you're going to have to come up with something very crafty here to reduce the overhead worst case." ([message](https://www.postgresql.org/message-id/CAApHDvpFKU6C3LNJZMF1cH6%2B-Ataf3oPBX3Dy7JH-q7kHb5EDA%40mail.gmail.com)) Different now: P2 skipped partition pruning in nodeNestloop.c when parameters did not change; this patch adds a gating Result above the inner side. Both have a measured worst case and no planner decision.

## Objections a reviewer is likely to raise

### 1. Unbounded worst-case cost with no planner choice to avoid it

A user runs a nested loop left join where the inner side is usually empty and the outer-only ON clause is expensive. After upgrade the gate runs on every outer row, and the query takes about three times longer. Nothing in the plan cost shows this, so the user finds out from production latency.

Message #21 measures about 1500 ms against about 470 ms on master and says the overhead is linear in outer size, with no ceiling. Yet v4 adds the gate at plan creation without costing. [P2] was returned for the same shape: a nestloop change with a worst-case slowdown and no planner or heuristic decision. Message #21 names the cost model as the open question, and message #19 notes a large footprint across regression test plans.

Precedent: [Improve the performance of nested loop join in the case of partitioned inner table](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com) (Returned with feedback, 2023-08-01)

**Question:** How does the planner avoid the gated plan when the inner side is empty and the gate is expensive, and what does it cost to model?

<sub>fixable, level 2, performance on the hot path; messages [#19](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com), [#21](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com); `prior-attempts-1`</sub>

### 2. Left joins that return rows today can raise errors after upgrade

An application runs `o left join i on o.k = i.k and 1/o.x = 1` against a table where `i` has no matching rows. On HEAD it gets `(1, 0, NULL)`. After upgrading to a release with this patch, the same query fails with division by zero, and the user finds out in production.

Message #22 shows this with v3, and message #23 calls it a false positive without saying why in the thread. Postgres does not promise an order of clause evaluation, so a committer may accept that a qual now runs where it did not before. But this patch evaluates the clause for every outer row, including rows where HEAD never reaches the join qual, and the planner makes this change for every eligible plan without a cost choice. The thread does not say whether v4 still gates clauses that can raise errors.

**Question:** Does v4 still gate clauses that can raise errors, and what is the in-thread argument that turning working queries into errors after upgrade is acceptable?

<sub>fixable, level 2, planner behaviour change; messages [#21](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com), [#22](https://www.postgresql.org/message-id/d510e2e0-c3aa-45f7-a8fb-a2684504aba3%40gmail.com), [#23](https://www.postgresql.org/message-id/297bc262-5d06-42c8-a319-555904e1bbbd%40gmail.com); `standards-compat-1`</sub>

### 3. Always-on plan change with no way to turn it off

After an upgrade, a reporting query with an expensive outer-only ON clause and a mostly empty inner side runs about three times slower, as in message #21. The DBA cannot disable the gate per session and has to rewrite the generated SQL, which an ORM such as Hibernate may not allow.

The gate is added at plan creation with no costing, and message #14 rejects a GUC. Recent planner features such as Memoize came with an enable_* setting, so users have an escape hatch when the new shape loses. Message #21 measures the loss: no upper bound, linear in outer size times gate cost. Message #19 says the change has a large footprint in the regression tests. That means many existing EXPLAIN outputs change shape, which also affects extension tests and pg_plan_advice users.

Precedent: [Improve the performance of nested loop join in the case of partitioned inner table](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com) (Returned with feedback, 2023-08-01)

**Question:** What can a user do after upgrade when the gated nested loop is slower for their query, without rewriting the SQL?

<sub>fixable, level 2, new GUC; messages [#14](https://www.postgresql.org/message-id/50b7ecd9-7512-479b-8096-2bda4e0e7ac1%40gmail.com), [#19](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com), [#21](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com); `standards-compat-2`</sub>

### 4. Extra plan node forces changes in plan-shape consumers

An extension or contrib module that matches nestloop inner plan shapes, such as pg_plan_advice or a third-party hint module, meets an unexpected Result node after upgrade. Its plan checks fail or its advice stops applying. The maintainers find out from user reports.

Patch 0002 exists only so that pg_plan_advice can look through the new Result node. Messages #18 and #19 call that fix "quite a formal fix" and say the correct way to extend this machinery is not yet known. Each later change to plan shape would need the same fix in every consumer. The proposal does not say how external modules should detect a gating Result.

**Question:** How should pg_plan_advice and other plan-shape consumers recognize a gating Result in general, beyond the special case that patch 0002 adds?

<sub>fixable, level 2, ABI / extension API; messages [#18](https://www.postgresql.org/message-id/bb73a0a0-70e6-46a1-900b-55dd75c2669d%40gmail.com), [#19](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. How does the planner avoid the gated plan when the inner side is usually empty and the outer-only clause is expensive?
2. Does v4 still gate clauses that can raise errors, such as `1/o.x = 1`, and what is the in-thread argument that the new error is acceptable?
3. What can a user do after upgrade when the gated nested loop is slower for a query that an ORM generates and that cannot be rewritten?
4. Why is gating at plan creation better than a cost-based GatingPath, given that nested loop cost estimates do not reflect the gate?
5. How should pg_plan_advice and other plan-shape consumers recognize a gating Result in general, beyond the special case in patch 0002?

## Touch points

- **ABI / extension API:** The patch adds a NestLoop flag in plannodes.h, and a second patch makes pg_plan_advice look through the new Result node.
- **new GUC:** The patch adds no GUC, and message #14 rejects one, so the always-on gate has no off switch.
- **planner behaviour change:** createplan.c adds a gating Result on the nested loop inner side for every eligible left join, without costing.
- **backpatchability:** see objection standards-compat-1: Left joins that return rows today can raise errors after upgrade
- **maintenance burden ("who maintains this in 10 years"):** The keep_inner_rewind flag, the changed EXPLAIN output in many regression tests, and the pg_plan_advice patch all need upkeep.
- **performance on the hot path:** The gate runs once per outer row in nodeNestloop.c; message #21 measures about 1500 ms against about 470 ms.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the planner's outer join clause placement and the nested loop rescan and Materialize code. Tom Lane and Andres Freund commented in 2024, but nobody of that kind has reviewed v1 to v4.
- Priority: fix design first; audience a sizeable minority; review cost medium. Hibernate treat() queries, a sales-schema request, and psql and pg_dump catalog queries show the pattern is common. The costing question and the new-error question are open, and the change touches the planner, the executor and many regression test plans.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
