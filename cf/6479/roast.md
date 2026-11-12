# Design pre-review: SLOPE - Planner optimizations on monotonic expressions.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6479](https://commitfest.postgresql.org/patch/6479/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxN8kLUMygXb33LC-UPGyTC0cTcCyeWUdfEoz%3DOBo3aT5g%40mail.gmail.com), 44 of 44 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal of grouping by time bucket with one timestamp index is not disputed, but level 1b is not settled. demand-1 shows the only stated use case is group by date_trunc or a date cast, while v16 annotates arithmetic, casts and timezone functions. maintenance-fit-3 and demand-2 show that every query pays planning cost, measured only for v5, and that no execution-time gain is shown. Three level 2 objections survived but are not evaluated here.

Not checked: The code was not read, so the plan-cache guard, the per-zone monotonicity check and the equalimage logic could not be checked. The three surviving level 2 objections (standards-compat-1, standards-compat-2, maintenance-fit-1) are not evaluated because level 1b fails first.

## Prior attempts

None found before 2026-02-11.

## Objections a reviewer is likely to raise

### 1. Demand is shown for time-bucket grouping only, but the scope is much wider

A committer takes on prosupport entries for multiply, divide, unary minus, atan, sinh, erf, casts and timezone conversions. Users who never group by f(x) pay the extra planning cost. Each later fix for a wrong-result corner case, like the infinity, domain, cast or DST cases, lands on that committer.

The only use case anyone stated is group by date_trunc or ts::date (messages #1, #3). In message #12 the author says the ORDER BY case is "a bit pointless" because order by x works, and that min(f(x)) can be written as f(min(x)). Most wrong-result reports in messages #7, #11, #23, #26 and #33 came from arithmetic, interval, cast and infinity coverage that no user asked for. The author offered to limit the scope to GROUP BY in message #12, but v16 keeps the full scope.

**Question:** Which users need ordering through arithmetic, trigonometric or cast expressions, and why would a version limited to date_trunc and date casts under GROUP BY not serve them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAE8JnxN8kLUMygXb33LC-UPGyTC0cTcCyeWUdfEoz%3DOBo3aT5g%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAE8JnxN_XzS1DgmEvEizbg7jXPgUwJ853uorciA5X8p%2B%3D9dYvA%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAE8JnxP__hhKjP8uXKqGepg%2BGAjRy%2BJthg%3Df-e24G3PML7ZdbA%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/CAN4CZFP%2BQkhGnt60_Z%2BQoaPj1nVfNtGFQhLtFxp_bTJ6SbMOMQ%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAN4CZFPhBcNP6jxA4u%3DV%3Dmie%3DDNyYe4t8cENCzXk49ojkZ%3D5JQ%40mail.gmail.com), [#33](https://www.postgresql.org/message-id/CAN4CZFNDVZm9wj86APOTfrx2aJrd6TEJfqiceEe0XMO%3Da81wEw%40mail.gmail.com); `demand-1`</sub>

### 2. Planning cost on every query for cases the author calls marginal

An OLTP workload with many indexes and ORDER BY on plain columns or casts pays extra planning time on every query. enable_slope is on by default, so these users pay a cost they never chose. The only numbers are from v5.

Message #10 shows about 5% overhead for ORDER BY a and 15% for casts on a table with 24 indexes, and v16 has much more logic than v5. In message #12 the author calls the ORDER BY and min/max cases "a bit pointless", because a rewrite gets the same plan, and offers to limit the analysis to GROUP BY. The full scope stayed (message #15), so all queries carry the cost of the broadest version.

**Question:** What are the v16 planning-time numbers for queries that gain nothing, and would a GROUP BY-only scope keep most of the benefit for less cost?

<sub>fixable, level 1b, performance on the hot path; messages [#10](https://www.postgresql.org/message-id/CAE8JnxOiw8B1FXb-fBGO-03xpVtZuZubCbpoW9LytmfFNB35Lw%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAE8JnxP__hhKjP8uXKqGepg%2BGAjRy%2BJthg%3Df-e24G3PML7ZdbA%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAE8JnxP75xOK9K7F0c74C9j4wnGEz9%2BxsuvwehExb3Lth3%2B3-A%40mail.gmail.com); `maintenance-fit-3`</sub>

### 3. No measured gain for the motivating group by query

A dashboard runs select date_trunc('month', ts), count(*) over a whole busy table. The planner now chooses a full index-only scan plus GroupAggregate, where today it would use a seq scan plus HashAggregate with a few groups. No run shows which plan is faster.

The thread shows EXPLAIN output and planning-time overhead from v5 (message #10), but no execution times before and after. For a full-table aggregate with few groups, today's hash plan may already be cheap, so the benefit likely depends on LIMIT, range filters or many groups. No one in the thread reports a field case or an independent user who hit this problem. Corey's "That's a good one" (message #4) is interest, not a report of need.

**Question:** For which query shapes and table sizes does the new plan beat today's seq scan plus HashAggregate, and by how much in execution time?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/CAE8JnxN_XzS1DgmEvEizbg7jXPgUwJ853uorciA5X8p%2B%3D9dYvA%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CADkLM%3Dc6-UuLNpN_O6e0Cuv7RWg1jS0bOd5WK_AJuPzR7JAD3Q%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAE8JnxOiw8B1FXb-fBGO-03xpVtZuZubCbpoW9LytmfFNB35Lw%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. What are the v16 planning-time numbers for queries that gain nothing from slope analysis, on tables with many indexes?
2. For which query shapes and table sizes does the new plan beat today's seq scan plus HashAggregate in execution time, and by how much?
3. Which users need ordering through arithmetic, trigonometric or cast expressions, and would a version limited to date_trunc and date casts under GROUP BY serve the stated use case?
4. Which plan-caching paths (PL/pgSQL, SPI, extended protocol, force_generic_plan) does the guard cover, and what invalidates a cached plan when TimeZone changes?
5. What mechanism, beyond regression tests for known cases, would catch a wrong or stale monotonicity annotation before it ships?

## Touch points

- **catalog changes:** Patches 0003 and 0007 add seven support functions and attach prosupport entries to many pg_proc rows.
- **ABI / extension API:** The patch adds SupportRequestMonotonic and changes the DecodeTimezoneName signature.
- **new GUC:** Patch 0004 adds enable_slope, default on.
- **planner behaviour change:** build_index_pathkeys emits extra pathkeys, so plans for ORDER BY, GROUP BY, DISTINCT and min/max change.
- **maintenance burden ("who maintains this in 10 years"):** Each monotonicity annotation is a correctness promise, and the patches couple the planner to src/timezone and session TimeZone.
- **performance on the hot path:** Slope analysis runs at plan time for every query; v5 numbers show 1% to 15% planning overhead with 24 indexes.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pathkeys and equivalence classes in the planner, and one who knows the datetime and timezone code. Dean Rasheed commented early and asked to defer; no such committer has reviewed the design since.
- Priority: needs design rethink; audience a sizeable minority; review cost high. Grouping by time bucket is a common report pattern, but v16 is eight patches across the planner, pg_proc, datetime and src/timezone, and its scope exceeds the stated demand. A narrower version with v16 measurements would be cheaper to review and own; patches 0002 and 0005 look separable and could go first.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 4.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
