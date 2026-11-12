# Design pre-review: Optimize mergejoinscansel() by avoiding redundant computations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7499](https://commitfest.postgresql.org/patch/7499/) (Needs review); authors: Zizhuan Liu (liuzizhuan)
- Thread: [Optimize mergejoinscansel() by avoiding redundant computations](https://www.postgresql.org/message-id/flat/tencent_E9D4668DEAF0286DAEB4FD13B3DF08961908%40qq.com), started 2026-10-22
- Messages read: 5 of 5
- Patch prose read: v4-0001-Skip-redundant-work-in-mergejoinscansel-for-equiv.patch, v1-0001-Optimize-mergejoinscansel-by-avoiding-redundant-c.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The proposal is clear and claims unchanged estimates, but demand-1 survives at level 1b. Every number comes from synthetic queries on copies of pg_class on one laptop, and the saving is about 0.05 ms per qualifying statement. The author asks in the thread whether the gain is worth it, and nobody has answered. The verdict rests on missing evidence in a thread with one participant, so confidence is low.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Level 2 was not evaluated; one design objection about drift between the equivalence rules and the estimator functions survived and waits there. Whether the rules keep estimates unchanged with stats hooks or pulled-up subqueries cannot be judged without the code.

## The proposal, restated

**Problem.** When both inputs of a merge clause are the same column or expression of the same relation, mergejoinscansel() computes the same ranges and estimates twice and then discards them for the default result. This costs planning time in self-joins and joins against pulled-up subqueries and views, since scalarineqsel() may probe an index for the actual min/max.

**Proposed design.** A new mergejoin_vardata_equivalent() runs after examine_variable() and decides whether the two inputs must give identical estimates. If so, and the operator is not cross-type, mergejoinscansel() skips get_variable_range() and scalarineqsel() and returns the default, adding stanullfrac for nulls-first when a range exists. Assert-enabled builds repeat the full computation via mergejoinscansel_internal() and check that both results match.

**User-visible surface.**
- get_variable_range() in selfuncs.c now accepts NULL for min and max, and then only reports whether a range is available.

**What the author claims.**
- Estimates and plans are unchanged; 29 merge join test cases give identical EXPLAIN output, costs included.
- In steady state, mergejoinscansel() is about 96% faster and planning time about 20% lower when the shortcut applies.
- When the shortcut does not apply, the extra check is lost in noise; one cold-connection case was about 0.3 ms slower, attributed to cache warming moving.
- Regression, isolation and most TAP tests pass, and the assert cross-check caught an injected bug.

**Thread state.** Only the author has posted, across five messages and four patch versions; no other reviewer has replied. The author asked for feedback on the correctness of the equivalence conditions and whether the gain is worth it.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-22; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No evidence that same-column merge self-joins cost real users planning time

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer takes on new equivalence logic in selfuncs.c to save about 0.05 ms per qualifying statement. Afterwards, no user can be shown whose workload plans many same-column merge self-joins where this time matters.

Every number comes from synthetic queries on copies of pg_class, run on one laptop under WSL1 (messages #1, #3, #5). Message #1 calls the pattern "common" but cites no field report, -general or -bugs thread, or production profile. The author asks in messages #1 and #5 whether the gain justifies the work, and no one else has replied. The 20% saving applies only to statements that are already cheap to plan, about 0.245 ms.

Thread: [#1 (ZizhuanLiu X-MAN, 2026-10-22)](https://www.postgresql.org/message-id/tencent_E9D4668DEAF0286DAEB4FD13B3DF08961908%40qq.com), [#3 (ZizhuanLiu X-MAN, 2026-10-25)](https://www.postgresql.org/message-id/tencent_F24ED47E73593650CB349EC2AFB775D0FE0A%40qq.com), [#5 (ZizhuanLiu X-MAN, 2026-10-30)](https://www.postgresql.org/message-id/tencent_11DC44DCF7BD6509754F633CFC4B1B05AD06%40qq.com)

**Question:** Which real workloads, such as ORM-generated views or reporting queries, spend measurable planning time on same-column merge clauses, and how large is that share in a profile?

Counter-check: held. The material shows only synthetic queries on copies of pg_class on one laptop, with a saving of about 0.05 ms per statement. The brief lists real-workload frequency as unknown, and the author asks whether the gain is worth it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | get_variable_range() in selfuncs.c now accepts NULL for min and max; this widens the contract that extensions can call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch adds a shortcut in mergejoinscansel(); the author claims estimates and plans stay the same. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | mergejoin_vardata_equivalent() must stay in sync with what get_variable_range() and scalarineqsel() read; only an assert-only cross-check guards it. |
| performance on the hot path | **yes** | Every mergejoinscansel() call runs the new equivalence check, including calls where the shortcut does not apply. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No evidence that same-column merge self-joins cost real users planning time |

## 5. Questions the author must answer before anyone reviews code

1. Which real workloads spend measurable planning time on same-column merge clauses, and what share of planning time does mergejoinscansel() take in a profile of them?
2. Does the pointer equality test on statistics tuples still hold when a stats hook supplies the data for one or both sides?
3. Which regression tests make the assert cross-check fail if scalarineqsel() starts to depend on a field that mergejoin_vardata_equivalent() does not compare?
4. What does the extra check cost on other hardware, and with many merge clauses where the shortcut does not apply?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the selectivity estimation code in selfuncs.c and how examine_variable() fills VariableStatData would have to own it. Only the author has posted.
- **Liability:** The owner signs for the claim that the shortcut never changes an estimate, and for keeping the equivalence rules in sync with get_variable_range() and scalarineqsel() for as long as the shortcut exists. Release builds have no check.
- **Author follow-up:** The author posted four patch versions in five messages and wrote in message #1 that they will continue the work if the direction is found promising.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A profile or field report from a real workload, for example queries over views or ORM-generated self-joins, where same-column merge clauses take a measurable share of planning time. Numbers from more than one machine would also help.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is small and touches one file, but a reviewer must check the equivalence conditions against the estimator code. The material shows a saving of about 0.05 ms per qualifying statement and no workload that needs it.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-1` (fixable, level 2) Equivalence rules become a second copy of selectivity internals. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-2` Planner time saved is small and limited to one query shape. Refuted (not specific): Duplicate of demand-1 on the missing demand evidence. The remaining part, that eqjoinsel() or other estimators would gain similar shortcuts, is speculation; the proposal does not suggest that.
- `demand-2` Small gain for one query shape adds a lasting coupling for selfuncs.c. Refuted (not specific): Duplicate of maintenance-fit-1: the failure scenario is the same drift between the equivalence rules and scalarineqsel(). The question of gain against group size is covered by demand-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only row, [P11], is about removing a NULL check from clause_selectivity_ext(), a different idea. Its recorded reasons, an unmeasured gain and a narrower contract for extensions, do not apply here: this proposal shows measurements, and get_variable_range() now accepts more inputs (NULL min/max), so existing callers keep working. Message #1 cites CommitFest 7302 only as the author's other work, not as an earlier attempt at this idea.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, dump content or psql change, and claims unchanged estimates and plans. Its one interface change, get_variable_range() accepting NULL min and max, only widens the contract, so existing extension callers keep working. Whether the equivalence rules really keep estimates unchanged, for example with stats hooks or pulled-up subqueries, is a correctness question for the design angle. It cannot be checked without the code.
- replication-ondisk: 0 objection(s). The patch touches only planner estimation in selfuncs.c and adds no on-disk, WAL, catalog, wire protocol or replication state. The get_variable_range() change only widens what callers may pass, so existing calls keep working and extensions are not broken.
- security: 0 objection(s). The patch has no new privilege or exposure surface: no new object, setting, function or view, and the shortcut returns the default estimate without reading statistics. The proposal requires equal acl_ok and keeps the nulls-first stanullfrac rule, which leaves leakproofness and RLS behaviour as in master. Whether the NULL-min/max path of get_variable_range() keeps the operator security check depends on the code and cannot be judged here.
- maintenance-fit: 2 objection(s). The get_variable_range() change accepts NULL in addition to the old contract, so existing extension callers are not affected; no objection raised there.
- already-exists: 0 objection(s). No existing feature or extension does this job. The per-clause cache in cached_scansel() avoids repeat calls for one merge clause, but it does not skip the duplicate work inside one mergejoinscansel() call, and this planner-internal shortcut cannot be built as an extension.
- demand: 2 objection(s). Two objections are raised. Both are of the not-now kind: the material shows no demand beyond synthetic benchmarks, and no other reviewer has replied yet.

### Precedents retrieved

- P1: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P2: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant; reason summarised from its thread
- P3: [Revert: Remove useless self-joins](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d286d83c0eed695910cb20d970ea9bea2e5001) (revert, Reverted, 2024-05-06) - judged not relevant
- P4: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant; reason summarised from its thread
- P5: [Revert "Get rid of the "new" and "old" entries in a view's rangetable."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0e6d6d3c909e14eedd370dd77d7111c2ae24c02) (revert, Reverted, 2023-01-12) - judged not relevant
- P6: [Revert "Put enable_self_join_elimination into postgresql.conf.sample"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a1a7c5a7071c75676c15b26e242c7df17560bd1) (revert, Reverted, 2025-04-06) - judged not relevant
- P7: [Revert change to autodetect '__getopt_initialized' variable ... turns out glibc doesn't actually export this variable, so we need a different solution.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2b7206a9935c915155a2c71e4658638de98ad440) (revert, Reverted, 2001-10-20) - judged not relevant
- P8: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P9: [SLOPE - Planner optimizations on monotonic expressions.](https://commitfest.postgresql.org/patch/6653/) (cf, Withdrawn, 2026-07-20) - judged not relevant; reason summarised from its thread
- P10: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant; reason summarised from its thread
- P11: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - used in the review; reason summarised from its thread
- P12: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P13: [Skip a redundant singleton GROUP BY node](https://commitfest.postgresql.org/patch/7419/) (cf, Withdrawn, 2026-10-09) - judged not relevant; reason summarised from its thread
- P14: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P15: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P16: [Revert incorrect memory-conservation hack in inheritance_planner().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0f36045b2e3d037bb7647d84373404fa4ba9588) (revert, Reverted, 2011-01-13) - judged not relevant
- P17: [Optimize-transaction-waiting-during-logical-decoding](https://commitfest.postgresql.org/patch/5806/) (cf, Withdrawn, 2026-04-25) - judged not relevant
- P18: [Simplify some RI checks to reduce SPI overhead](https://commitfest.postgresql.org/patch/2948/) (cf, Returned with feedback, 2022-04-09) - judged not relevant
- P19: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant
- P20: [Plan a filtering inner join as a semijoin](https://commitfest.postgresql.org/patch/7202/) (cf, Withdrawn, 2026-08-27) - judged not relevant
- P21: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P22: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P24: [Optimize Numeric using fixed buffer reducing pallocs for small numbers](https://commitfest.postgresql.org/patch/4196/) (cf, Returned with feedback, 2023-09-13) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 115 s; cost reported by the CLI: $1.59
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 63 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:01:27.337Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
