# Design pre-review: Lazy LEFT NestLoop JOIN: gate rescanning of the inner side by outer-side-only join clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6981](https://commitfest.postgresql.org/patch/6981/) (Needs review); authors: Andrei Lepikhov (lepikhov), Mikhail Zhilin (mizhka-at-pgpro), Peter Petrov (pspetrov)
- Thread: [Do not scan index in right table if condition for left join evaluates to false using columns in left table](https://www.postgresql.org/message-id/flat/CAKE%3DrqQ-LHuh2eVsKC7ihkRJoCBZafSR72o3Xk4Xb%3DLcQMQfsA%40mail.gmail.com), started 2024-12-07
- Messages read: 23 of 23
- Patch prose read: v2-lazy-join.diff (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has shown demand, and the gating Result design follows what Tom Lane and Andres Freund suggested. Level 2 fails on open design points. prior-attempts-1: message #21 measures about 1500 ms against about 470 ms on master, and the planner makes no cost-based choice. standards-compat-1: a left join that returns rows on HEAD can raise division by zero, and the thread answers only with a link to a blog post. No reviewer has commented on opportunistic gating versus a cost-based path.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The v4 code was not read. I could not check which clauses v4 gates (error-raising, stable or non-leakproof), whether semi joins are affected, or how keep_inner_rewind interacts with Materialize rescans.

## The proposal, restated

**Problem.** In a nested loop left join, Postgres scans the inner side for every outer row, even when an ON clause that uses only outer columns is false. This hits ORM-generated polymorphic queries (Hibernate treat()) and sales schemas that left join product rows to per-category tables, and Andres Freund named psql and pg_dump catalog queries with the same pattern.

**Proposed design.** The current patch puts a gating Result node with a One-Time Filter above the inner side of a nested loop. The filter holds the join clauses that reference only the outer relation, so the inner subtree runs only when they pass. The change is made at plan creation with no cost-based choice, adds a NestLoop flag 'keep_inner_rewind' so a Materialize on the inner side is not rescanned, and adds a second patch for pg_plan_advice.

**User-visible surface.**
- EXPLAIN shows a new Result node with One-Time Filter on the inner side of a nested loop left join for outer-only ON clauses.
- pg_plan_advice: a separate patch makes it look through gating Result nodes on the nested loop inner side.
- Regression tests: patch 0001 adds a shared platform_independent_explain routine to mask platform-dependent EXPLAIN output.

**What the author claims.**
- In the products/phones/vehicles example, each inner index scan runs about 5,000 times instead of 10,036.
- A gated nested loop adds no noticeable overhead when rows pass the gate, so the author sees no need for a GUC or costing.
- Worst case: an expensive gate with an empty inner side took about 1500 ms, against about 470 ms on master; overhead grows linearly with outer size.
- v4 fixes wrong row counts with volatile clauses by following the qual_is_pushdown_safe rules.
- The division-by-zero difference Bryan Green reported is a false positive, explained in the author's blog post.

**Thread state.** Tom Lane and Andres Freund doubted in 2024 that the case is worth new mechanism, but suggested the gating Result approach the patch now uses. Since 2026 only Bryan Green has replied, reporting wrong results; v4 addresses the volatile cases, the division-by-zero case is disputed, and CFbot reports the patch needs a rebase.

## 2. Prior attempts

### Improve the performance of nested loop join in the case of partitioned inner table (related)

- **Link:** [https://commitfest.postgresql.org/patch/4253/](https://commitfest.postgresql.org/patch/4253/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com)
- **Date:** decided 2023-08-01; started 2023-03-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** David Rowley (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author's own benchmark showed about a 4% slowdown when the parameter changes on every outer row, against about 20% gain when it stays the same. David Rowley judged that changed parameters are the common case and asked for a planner decision based on n_distinct or an executor heuristic, with the logic moved out of nodeNestloop.c. The author did not post an update, so the entry was returned with feedback.
- **Reason (quote):** "In short, I think you're going to have to come up with something very crafty here to reduce the overhead worst case." ([message](https://www.postgresql.org/message-id/CAApHDvpFKU6C3LNJZMF1cH6%2B-Ataf3oPBX3Dy7JH-q7kHb5EDA%40mail.gmail.com))
- **What is different this time:** P2 skipped partition pruning in nodeNestloop.c when parameters did not change; this patch adds a gating Result above the inner side. Both have a measured worst case and no planner decision.


## 3. Objections a reviewer is likely to raise

### 1. Unbounded worst-case cost with no planner choice to avoid it

fixable · level 2, rubric item 8 · performance on the hot path, planner behaviour change · `prior-attempts-1`

A user runs a nested loop left join where the inner side is usually empty and the outer-only ON clause is expensive. After upgrade the gate runs on every outer row, and the query takes about three times longer. Nothing in the plan cost shows this, so the user finds out from production latency.

Message #21 measures about 1500 ms against about 470 ms on master and says the overhead is linear in outer size, with no ceiling. Yet v4 adds the gate at plan creation without costing. [P2] was returned for the same shape: a nestloop change with a worst-case slowdown and no planner or heuristic decision. Message #21 names the cost model as the open question, and message #19 notes a large footprint across regression test plans.

**Precedents.**
- [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (Returned with feedback, 2023-08-01); thread: [link](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com)

Thread: [#19 (Andrei Lepikhov, 2026-07-28)](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com), [#21 (Andrei Lepikhov, 2026-09-12)](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com)

**Question:** How does the planner avoid the gated plan when the inner side is empty and the gate is expensive, and what does it cost to model?

Counter-check: held. Message #21 measures about 1500 ms against about 470 ms on master and leaves the cost model open. [P2] fits: a nested loop change with a measured worst case, where the reviewer asked for a planner decision.

### 2. Left joins that return rows today can raise errors after upgrade

fixable · level 2, rubric item 7 · planner behaviour change, backpatchability · `standards-compat-1`

An application runs `o left join i on o.k = i.k and 1/o.x = 1` against a table where `i` has no matching rows. On HEAD it gets `(1, 0, NULL)`. After upgrading to a release with this patch, the same query fails with division by zero, and the user finds out in production.

Message #22 shows this with v3, and message #23 calls it a false positive without saying why in the thread. Postgres does not promise an order of clause evaluation, so a committer may accept that a qual now runs where it did not before. But this patch evaluates the clause for every outer row, including rows where HEAD never reaches the join qual, and the planner makes this change for every eligible plan without a cost choice. The thread does not say whether v4 still gates clauses that can raise errors.

Thread: [#21 (Andrei Lepikhov, 2026-09-12)](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com), [#22 (Bryan Green, 2026-09-14)](https://www.postgresql.org/message-id/d510e2e0-c3aa-45f7-a8fb-a2684504aba3%40gmail.com), [#23 (Andrei Lepikhov, 2026-09-15)](https://www.postgresql.org/message-id/297bc262-5d06-42c8-a319-555904e1bbbd%40gmail.com)

**Question:** Does v4 still gate clauses that can raise errors, and what is the in-thread argument that turning working queries into errors after upgrade is acceptable?

Counter-check: held. Postgres does not promise a clause evaluation order, which weakens this objection. But message #23 answers the report only with a link to a blog post, and the thread does not say whether v4 still gates clauses that can raise errors.

### 3. Always-on plan change with no way to turn it off

fixable · level 2, rubric item 12 · new GUC, planner behaviour change · `standards-compat-2`

After an upgrade, a reporting query with an expensive outer-only ON clause and a mostly empty inner side runs about three times slower, as in message #21. The DBA cannot disable the gate per session and has to rewrite the generated SQL, which an ORM such as Hibernate may not allow.

The gate is added at plan creation with no costing, and message #14 rejects a GUC. Recent planner features such as Memoize came with an enable_* setting, so users have an escape hatch when the new shape loses. Message #21 measures the loss: no upper bound, linear in outer size times gate cost. Message #19 says the change has a large footprint in the regression tests. That means many existing EXPLAIN outputs change shape, which also affects extension tests and pg_plan_advice users.

**Precedents.**
- [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (Returned with feedback, 2023-08-01); thread: [link](https://www.postgresql.org/message-id/flat/CAHdr9ueefdCCCq1mXAYfLmRgEKfhWQDCU7=XmureEAzJcOwfag@mail.gmail.com)

Thread: [#14 (Andrei Lepikhov, 2024-12-09)](https://www.postgresql.org/message-id/50b7ecd9-7512-479b-8096-2bda4e0e7ac1%40gmail.com), [#19 (Andrei Lepikhov, 2026-07-28)](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com), [#21 (Andrei Lepikhov, 2026-09-12)](https://www.postgresql.org/message-id/2c0c380b-fab0-419e-a183-ab08ce2ee1ea%40gmail.com)

**Question:** What can a user do after upgrade when the gated nested loop is slower for their query, without rewriting the SQL?

Counter-check: held. Message #14 rejects a GUC because of low overhead, but message #21 later measures a threefold slowdown, so the thread does not answer this. [P2] is not about an off switch and must not be used as support.

### 4. Extra plan node forces changes in plan-shape consumers

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

An extension or contrib module that matches nestloop inner plan shapes, such as pg_plan_advice or a third-party hint module, meets an unexpected Result node after upgrade. Its plan checks fail or its advice stops applying. The maintainers find out from user reports.

Patch 0002 exists only so that pg_plan_advice can look through the new Result node. Messages #18 and #19 call that fix "quite a formal fix" and say the correct way to extend this machinery is not yet known. Each later change to plan shape would need the same fix in every consumer. The proposal does not say how external modules should detect a gating Result.

Thread: [#18 (Andrei Lepikhov, 2026-07-03)](https://www.postgresql.org/message-id/bb73a0a0-70e6-46a1-900b-55dd75c2669d%40gmail.com), [#19 (Andrei Lepikhov, 2026-07-28)](https://www.postgresql.org/message-id/ad949722-6adb-4c11-bd7c-33ac1db29ed6%40gmail.com)

**Question:** How should pg_plan_advice and other plan-shape consumers recognize a gating Result in general, beyond the special case that patch 0002 adds?

Counter-check: held. Gating Result nodes already exist in plans, so the point about third-party modules applies to any plan-shape change. But message #19 calls the pg_plan_advice fix formal and says the correct way to extend it is not yet known.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a NestLoop flag in plannodes.h, and a second patch makes pg_plan_advice look through the new Result node. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds no GUC, and message #14 rejects one, so the always-on gate has no off switch. |
| planner behaviour change | **yes** | createplan.c adds a gating Result on the nested loop inner side for every eligible left join, without costing. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Left joins that return rows today can raise errors after upgrade |
| maintenance burden ("who maintains this in 10 years") | **yes** | The keep_inner_rewind flag, the changed EXPLAIN output in many regression tests, and the pg_plan_advice patch all need upkeep. |
| performance on the hot path | **yes** | The gate runs once per outer row in nodeNestloop.c; message #21 measures about 1500 ms against about 470 ms. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does the planner avoid the gated plan when the inner side is usually empty and the outer-only clause is expensive?
2. Does v4 still gate clauses that can raise errors, such as `1/o.x = 1`, and what is the in-thread argument that the new error is acceptable?
3. What can a user do after upgrade when the gated nested loop is slower for a query that an ORM generates and that cannot be rewritten?
4. Why is gating at plan creation better than a cost-based GatingPath, given that nested loop cost estimates do not reflect the gate?
5. How should pg_plan_advice and other plan-shape consumers recognize a gating Result in general, beyond the special case in patch 0002?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the planner's outer join clause placement and the nested loop rescan and Materialize code. Tom Lane and Andres Freund commented in 2024, but nobody of that kind has reviewed v1 to v4.
- **Liability:** The owner signs for wrong-result and new-error reports on left joins, for plan regressions with expensive outer-only clauses, and for keeping keep_inner_rewind consistent with later rescan changes, with no end date.
- **Author follow-up:** The author posted v4 to fix the wrong row counts that Bryan Green reported and moved the entry to the next CommitFest in October 2026. CFbot has reported a pending rebase since 2026-10-20.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Hibernate treat() queries, a sales-schema request, and psql and pg_dump catalog queries show the pattern is common. The costing question and the new-error question are open, and the change touches the planner, the executor and many regression test plans.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Gate is applied without costing and has an unbounded worst case. Refuted (not specific): Duplicate of prior-attempts-1. It has the same measurement, the same precedent and the same request for a cost-based choice.
- `maintenance-fit-2` New nestloop kind between parameterized and plain adds a silent trap. Refuted (no failure scenario): The scenario depends on an unnamed future patch and on the assumption that no test covers the rescan count. Message #17 shows the regression in EXPLAIN ANALYZE loop counts on regression tables, which a test can pin.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent is the same idea. The thread's own 2024 discussion (messages #3, #4, #7) doubted demand but proposed the gating Result design that the patch now follows.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. The proposal adds no SQL syntax, catalog objects or pg_dump changes, so those touch points are clear.
- replication-ondisk: 0 objection(s). The proposal changes only plan shape and executor behaviour inside one backend: a gating Result node and a NestLoop flag. It adds no on-disk format, WAL record, catalog column, wire protocol message or replicated state. The plan node struct change goes only into a new major version and is not backpatched, so no stable ABI is affected. Effects on plan-walking extensions such as pg_plan_advice belong to other angles.
- security: 0 objection(s). I found no privilege or exposure aspect, so I raise no objections from the security angle. The patch adds no object, privilege, GUC, function or catalog entry. The gating Result runs on the inner side, after the outer scan has already applied its RLS and security barrier quals, so the gate's outer-only clause sees only rows the user can already read. It cannot see inner data, so it cannot leak inner rows past inner-side RLS. Bryan Green audited v3 for security in message #22 and reported only correctness differences. The extra error in that report (division by zero) and the volatile-clause row counts are correctness issues, and the slowdown with an expensive gate is a performance issue; other angles cover these. Without the code I cannot check whether v4 still evaluates leaky or non-leakproof clauses on rows the old plan never reached.
- maintenance-fit: 3 objection(s). Three objections are raised from the maintenance and fit angle.
- already-exists: 0 objection(s). Postgres has no existing feature that skips the inner scan of a nested loop left join for outer-only ON clauses in generated SQL. A hand-written lateral subquery fenced with OFFSET 0 can likely get a one-time filter, but the Hibernate treat() queries in message #5 cannot be rewritten, and no known extension does this.
- demand: 0 objection(s). The thread shows demand from three independent sources: Hibernate treat() queries (message #5), a sales-schema user request (message #8), and Postgres's own psql and pg_dump catalog queries (message #6). ORM-generated SQL cannot easily be rewritten, and the gate's cost falls only on queries that use the pattern, so I see no demand objection. Production numbers would still help the cost-model debate.

### Precedents retrieved

- P1: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P2: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - used in the review; reason summarised from its thread
- P3: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - used in the review; reason summarised from its thread
- P4: [INNER JOIN removals](https://commitfest.postgresql.org/patch/27/) (cf, Rejected, 2015-02-14) - judged not relevant; reason summarised from its thread
- P5: [Table Partition + Join Pushdown](https://commitfest.postgresql.org/patch/325/) (cf, Rejected, 2016-01-25) - judged not relevant; reason summarised from its thread
- P6: [hash joins with bloom filters](https://commitfest.postgresql.org/patch/1540/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P7: [INNER JOIN removals](https://commitfest.postgresql.org/patch/130/) (cf, Returned with feedback, 2015-05-11) - judged not relevant; reason summarised from its thread
- P8: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P9: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P10: [Remove LEFT JOINs in more cases](https://commitfest.postgresql.org/patch/1364/) (cf, Returned with feedback, 2018-09-28) - used in the review; reason summarised from its thread
- P11: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P12: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/20/) (cf, Rejected, 2015-01-16) - judged not relevant
- P13: [bloom filter in hash joins with batching](https://commitfest.postgresql.org/patch/446/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P14: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P15: [Revert my best_inner_indexscan patch of yesterday, which turns out to have had a bad side-effect:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a81e281636ac8c927528648d9eed04e8083fcdf5) (revert, Reverted, 2006-04-09) - judged not relevant
- P16: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Support run-time partition pruning for hash join](https://commitfest.postgresql.org/patch/4512/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P18: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - judged not relevant
- P19: [Hashjoin fall back to block nested hashloop join](https://commitfest.postgresql.org/patch/2588/) (cf, Withdrawn, 2020-11-10) - judged not relevant
- P20: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P21: [Fast path for empty relids in check_outerjoin_delay()](https://commitfest.postgresql.org/patch/1914/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P22: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P23: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant
- P24: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 151 s; cost reported by the CLI: $2.57
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 78 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:11:52.842Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
