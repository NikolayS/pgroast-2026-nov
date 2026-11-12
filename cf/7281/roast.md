# Design pre-review: Improve Hash/Merge Join estimate accuracy when all predicates are Hash/Merge clauses

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7281](https://commitfest.postgresql.org/patch/7281/); [thread](https://www.postgresql.org/message-id/flat/7abcff03-b38c-432f-81fa-d66abf0f061d%40tantorlabs.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the goal of more accurate merge and hash join costs is not disputed. The open points are fixable: demand-1 asks for evidence that plan choice improves, standards-compat-1 asks whether the "stable" target means back-patching, and maintenance-fit-2 notes that nobody who knows selectivity estimation has judged the tradeoff in 0001. That tradeoff, raised in message #6 and accepted in message #7, needs a committer who knows the area.

Not checked: Without the code, the review could not check that 0002's condition matches exactly the joins whose only join clauses are the merge or hash clauses. It also could not check how 0001 detects strict restriction clauses and combines null fractions.

## Prior attempts

None found before 2025-07-10.

## Objections a reviewer is likely to raise

### 1. No real query shown where the cost change alters the chosen plan

After upgrade, a user with a stable workload sees some inner merge or hash joins switch to a different plan, because their run cost rose by cpu_tuple_cost per extra tuple. The material does not show whether these switches make queries faster or slower, so users find out from changed query times in production.

Every example in the thread is synthetic and turns off enable_hashjoin and enable_nestloop, so the plan never changes (messages #3, #4, #6). The 0002 delta in message #3 is 800 on a cost of about 10,900, and no benchmark shows that the change picks a better plan. 0002 applies to all plain inner merge and hash joins, so every user takes the risk of plan changes. Message #7 accepts that 0001 can underestimate when filters correlate with NULLs, which adds to that risk.

**Question:** Is there a real or benchmark workload, with planner settings left at their defaults, where either patch changes the chosen plan and the query time improves?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/d5c69231-5669-4d5b-a821-0b97f102a5ae%40tantorlabs.com), [#4](https://www.postgresql.org/message-id/179051307018.1124.2392986466048370734.pgcf%40coridan.postgresql.org), [#6](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com); `demand-1`</sub>

### 2. Target "stable" would ship planner estimate changes in minor releases

A site applies a minor update on a branch where 0001 was back-patched. A query with a filter correlated with FK nullness, as in message #6, has its join estimate halved. The planner then picks a nested loop or a different join order, and the site finds a slower query in production with no change of its own to explain it.

The CommitFest entry targets "stable", and nothing in the thread says whether back-patching is intended. Both patches change costs, and 0001 also changes join row estimates that feed every plan above the join. Message #7 accepts a known underestimate when NULLs correlate with a filter. Back branches normally avoid estimate changes that can flip plans without the user asking, so a committer would likely ask that the target be master only.

**Question:** Is this series meant for master only, and if not, why should known plan-changing estimate shifts go into minor releases?

<sub>fixable, level 1b, backpatchability; messages [#6](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com); `standards-compat-1`</sub>

### 3. No one familiar with selectivity estimation has reviewed the FK change

0001 is committed with its independence assumption between FK nullness and other filters. Users whose filters correlate with NULL FK values see join estimates halve, as in message #6, and get worse plans. The committer who pushed it owns these reports without an expert to judge whether the tradeoff was right.

Both participants ask for review from someone who knows selfuncs and cost estimation (messages #6, #7), and none has replied. The author agrees that 0001 swaps one assumption for another and that the scope may need to be narrower (message #7). FK-based join selectivity has been reverted before over bugs found after commit [P1], so changes in this function need an owner who knows the area.

Precedent: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (Reverted, 2016-06-07)

**Question:** Has a committer who knows get_foreign_key_join_selectivity() reviewed whether the table-wide null fraction is an acceptable tradeoff against the case in message #6?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/tencent_BCBE9B4D2895C83674CF7C343A11B7AC1208%40qq.com), [#7](https://www.postgresql.org/message-id/CAAgWR6Zm9YKHmYrF--BGOneBbXz1MaPgPFncbKFAdaz0vRSQsw%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Is there a workload, with planner settings at their defaults, where either patch changes the chosen plan and the query time improves?
2. Is the series meant for master only, and if not, why should plan-changing estimate shifts go into minor releases?
3. Should 0001 apply only when the referencing rel has no restriction clauses, given the underestimate shown in message #6?
4. Which regression outputs change with v4, given that v1 changed several expected files and v4 touches only costsize.c?
5. With MATCH SIMPLE and partly NULL multi-column FKs, how far can the largest per-column null fraction be from the true fraction of non-matching rows?

## Touch points

- **planner behaviour change:** 0002 changes merge and hash join run costs, and 0001 changes FK join row estimates, so plan choice can change.
- **backpatchability:** The CommitFest entry targets "stable", and the thread does not say whether back-patching is intended.
- **maintenance burden ("who maintains this in 10 years"):** 0001 replaces one estimation assumption in get_foreign_key_join_selectivity() with another, and no area expert has reviewed it.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No real query shown where the cost change alters the chosen plan

## Who would commit it, and when

- Owner: silent. Someone who knows join selectivity estimation and costsize.c, in particular get_foreign_key_join_selectivity() and the merge and hash join cost functions.
- Priority: needs champion; audience a sizeable minority; review cost medium. Multi-column and nullable FK joins are common, and both patches are small and touch only costsize.c. Both participants ask for review by someone who knows selfuncs, and none has replied; review cost is set to medium because no code was read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
