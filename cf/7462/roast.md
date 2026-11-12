# Design pre-review: Improve ndistinct estimates for flattened UNION

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7462](https://commitfest.postgresql.org/patch/7462/); [thread](https://www.postgresql.org/message-id/flat/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Nothing that survived blocks the idea. The parent columns of a flattened UNION ALL get the default of 200 today, and the cost falls only on queries of that shape. Two fixable points remain: demand-1 (no overlapping-arm case is measured, so hi may overstate group counts) and maintenance-fit-1 (a new caller of get_variable_numdistinct() gets lo without choosing it). A planner committer has to judge the lo/hi policy.

Not checked: The code was not read. This review could not check how the lo/hi choice reaches each caller, whether the member lookups keep the usual statistics privilege checks, or what the on-demand traversal costs for wide unions.

## Prior attempts

None found before 2026-10-15.

## Objections a reviewer is likely to raise

### 1. Benefit is shown only on synthetic disjoint-key scenarios

A common pattern is a UNION ALL of current and archive tables whose keys overlap heavily. With the patch, the planner uses hi and estimates far more groups than exist. It may then change aggregation or join plans, and the user notices only when a query that was fine before slows down after upgrade.

All evidence in message #1 comes from the author's own 18-scenario matrix and reproducers built around disjoint arms; no field report or user story is cited. The Memoize motivation changed after message #2, and message #3 offers only new author-built cases. The problem itself is plausible, but which real query shapes gain, and how often arms overlap, decides whether the change helps most users or just moves the error.

**Question:** Which real workloads, from users or bug reports, hit the DEFAULT_NUM_DISTINCT estimate on UNION ALL columns, and how much do their arms overlap?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com), [#2](https://www.postgresql.org/message-id/CAApHDvqjEiQ9h9DwTnwtm30cUk9Ad7R3C3vLCZj%2BdOTTz-UB-Q%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/tencent_AF4D43955D36DFCFCF34FE117DAC0ABEB008%40qq.com); `demand-1`</sub>

### 2. Two ndistinct ends make every future consumer choose, and nothing checks

A later patch adds a planner consumer that treats ndistinct as an output cardinality and calls get_variable_numdistinct(). For UNION ALL parents it silently gets lo, the largest single member, and undercounts groups. No test or assertion fails, so the bad estimate surfaces only as a user's slow plan report.

Patch 0001 returns lo from get_variable_numdistinct() by default. Patch 0002 adds hi only through consumer-specific rules in selfuncs.c for GROUP BY, DISTINCT, Memoize and semi/anti joins. The correct end now depends on how each caller uses the number, and the proposal adds no API shape that forces a caller to state that role. Message #1 calls this split "consumer-specific safety policy", which a reviewer of an unrelated future patch is likely to miss.

**Question:** How does the API make a new caller pick lo or hi explicitly, so that a caller using the count as a cardinality cannot get lo by default?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/tencent_D4B30A0D790430EB89AF4E537FE59E4DD106%40qq.com); `maintenance-fit-1`</sub>

## Questions for the author

1. What estimates and plans does the patch give for GROUP BY, DISTINCT and Memoize when the UNION ALL arms overlap heavily, for example a current table and an archive table?
2. How does the API make a new caller pick lo or hi explicitly, so that a caller using the count as a cardinality cannot get lo by default?
3. Which real workloads, from users or bug reports, hit the DEFAULT_NUM_DISTINCT estimate on UNION ALL columns?
4. What are the planning-time numbers for wide unions, such as the 64-arm cases, before and after the patch?
5. Are inheritance and partitioned appendrels or non-flattened UNION ALL subqueries affected in any way, and if not, why is flattened UNION ALL the right scope?

## Touch points

- **ABI / extension API:** Patch 0001 changes pathnodes.h, paths.h and selfuncs.h, and get_variable_numdistinct() now returns lo for flattened UNION ALL parents.
- **planner behaviour change:** Semi/anti join selectivity, GROUP BY/DISTINCT group counts and Memoize costing change for flattened UNION ALL parents.
- **maintenance burden ("who maintains this in 10 years"):** Each consumer of ndistinct must now pick lo or hi, through per-consumer rules in selfuncs.c.
- **performance on the hot path:** The planner walks member estimates lazily; message #1 reports planning-time increases in 64-arm cases, not statistically significant.

## Who would commit it, and when

- Owner: qualified committer engaged. David Rowley replied in message #2, but only on the Memoize claim. A committer who knows selfuncs.c and the appendrel planning code would have to sign off on the lo/hi policy.
- Priority: proceed; audience a sizeable minority; review cost medium. The patch adds no user-visible surface and changes estimates only for flattened simple UNION ALL. The two patches are small but change selfuncs.c and allpaths.c, and the lo/hi policy needs a planner reviewer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
