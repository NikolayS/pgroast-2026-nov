# Design pre-review: Optimize mergejoinscansel() by avoiding redundant computations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7499](https://commitfest.postgresql.org/patch/7499/); [thread](https://www.postgresql.org/message-id/flat/tencent_E9D4668DEAF0286DAEB4FD13B3DF08961908%40qq.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The proposal is clear and claims unchanged estimates, but demand-1 survives at level 1b. Every number comes from synthetic queries on copies of pg_class on one laptop, and the saving is about 0.05 ms per qualifying statement. The author asks in the thread whether the gain is worth it, and nobody has answered. The verdict rests on missing evidence in a thread with one participant, so confidence is low.

Not checked: Level 2 was not evaluated; one design objection about drift between the equivalence rules and the estimator functions survived and waits there. Whether the rules keep estimates unchanged with stats hooks or pulled-up subqueries cannot be judged without the code.

## Prior attempts

None found before 2026-10-22.

## Objections a reviewer is likely to raise

### 1. No evidence that same-column merge self-joins cost real users planning time

A committer takes on new equivalence logic in selfuncs.c to save about 0.05 ms per qualifying statement. Afterwards, no user can be shown whose workload plans many same-column merge self-joins where this time matters.

Every number comes from synthetic queries on copies of pg_class, run on one laptop under WSL1 (messages #1, #3, #5). Message #1 calls the pattern "common" but cites no field report, -general or -bugs thread, or production profile. The author asks in messages #1 and #5 whether the gain justifies the work, and no one else has replied. The 20% saving applies only to statements that are already cheap to plan, about 0.245 ms.

**Question:** Which real workloads, such as ORM-generated views or reporting queries, spend measurable planning time on same-column merge clauses, and how large is that share in a profile?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/tencent_E9D4668DEAF0286DAEB4FD13B3DF08961908%40qq.com), [#3](https://www.postgresql.org/message-id/tencent_F24ED47E73593650CB349EC2AFB775D0FE0A%40qq.com), [#5](https://www.postgresql.org/message-id/tencent_11DC44DCF7BD6509754F633CFC4B1B05AD06%40qq.com); `demand-1`</sub>

## Questions for the author

1. Which real workloads spend measurable planning time on same-column merge clauses, and what share of planning time does mergejoinscansel() take in a profile of them?
2. Does the pointer equality test on statistics tuples still hold when a stats hook supplies the data for one or both sides?
3. Which regression tests make the assert cross-check fail if scalarineqsel() starts to depend on a field that mergejoin_vardata_equivalent() does not compare?
4. What does the extra check cost on other hardware, and with many merge clauses where the shortcut does not apply?

## Touch points

- **ABI / extension API:** get_variable_range() in selfuncs.c now accepts NULL for min and max; this widens the contract that extensions can call.
- **planner behaviour change:** The patch adds a shortcut in mergejoinscansel(); the author claims estimates and plans stay the same.
- **maintenance burden ("who maintains this in 10 years"):** mergejoin_vardata_equivalent() must stay in sync with what get_variable_range() and scalarineqsel() read; only an assert-only cross-check guards it.
- **performance on the hot path:** Every mergejoinscansel() call runs the new equivalence check, including calls where the shortcut does not apply.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No evidence that same-column merge self-joins cost real users planning time

## Who would commit it, and when

- Owner: silent. Someone who knows the selectivity estimation code in selfuncs.c and how examine_variable() fills VariableStatData would have to own it. Only the author has posted.
- Not now, show demand first: A profile or field report from a real workload, for example queries over views or ORM-generated self-joins, where same-column merge clauses take a measurable share of planning time. Numbers from more than one machine would also help.
- Priority: park; audience unclear; review cost medium. The patch is small and touches one file, but a reviewer must check the equivalence conditions against the estimator code. The material shows a saving of about 0.05 ms per qualifying statement and no workload that needs it.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
