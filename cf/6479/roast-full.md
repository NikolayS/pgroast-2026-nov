# Design pre-review: SLOPE - Planner optimizations on monotonic expressions.

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6479](https://commitfest.postgresql.org/patch/6479/) (Needs review); authors: Alexandre Felipe (afelipe)
- Thread: [SLOPE - Planner optimizations on monotonic expressions.](https://www.postgresql.org/message-id/flat/CAE8JnxN8kLUMygXb33LC-UPGyTC0cTcCyeWUdfEoz%3DOBo3aT5g%40mail.gmail.com), started 2026-02-11
- Messages read: 44 of 44
- Patch prose read: v16.0-0003-SLOPE-catalog-changes.patch, v16.0-0001-benchmark.patch, v16.0-0002-Optimized-reverse-pathkeys.patch, v16.0-0005-SLOPE-Constant-sort-keys.patch, v16.0-0004-SLOPE-Planner-support.patch, v16.0-0007-SLOPE-Timezone.patch, v16.0-0006-SLOPE-documentation.patch, v16.0-0008-SLOPE-timezone-errsave.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal of grouping by time bucket with one timestamp index is not disputed, but level 1b is not settled. demand-1 shows the only stated use case is group by date_trunc or a date cast, while v16 annotates arithmetic, casts and timezone functions. maintenance-fit-3 and demand-2 show that every query pays planning cost, measured only for v5, and that no execution-time gain is shown. Three level 2 objections survived but are not evaluated here.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the plan-cache guard, the per-zone monotonicity check and the equalimage logic could not be checked. The three surviving level 2 objections (standards-compat-1, standards-compat-2, maintenance-fit-1) are not evaluated because level 1b fails first.

## The proposal, restated

**Problem.** The planner does not know that f(x) is ordered when x is ordered, so queries such as group by date_trunc('month', ts) or created_at::date sort or read the whole table instead of using an index on the timestamp column. The author targets reports and dashboards on busy tables that want one timestamp index for several grouping intervals.

**Proposed design.** A new prosupport request lets a function declare, per argument, whether it is increasing, decreasing, constant or neither. At plan creation the planner finds the innermost subexpression that causes all variation of each pathkey expression and caches it as slope_info. build_index_pathkeys then emits extra pathkeys when the index key is that source, using a backward scan for decreasing functions.

**User-visible surface.**
- New GUC enable_slope (boolean, default on).
- New support request SupportRequestMonotonic, using MONOTONICFUNC_INCREASING, _DECREASING, _BOTH and _NONE values, for C extension functions.
- New built-in support functions arg0_asc_slope_support, arg0_desc_slope_support, arg1_asc_slope_support, diff_slope_support, addition_slope_support, multiply_slope_support, divide_slope_support, attached to many pg_proc entries.
- DecodeTimezoneName gains an ErrorSaveContext parameter (C API change).
- btree equalimage functions may now be called at index scan planning time, per btree.sgml.

**What the author claims.**
- One timestamp index can serve group by on day, week, month or year, plus ordering, without a sort node.
- Min and max of f(x) can use an index scan with limit 1.
- Planning time overhead in v5 was about 1% to 15% on a table with 24 indexes, worst for casts.
- Computing backward pathkeys by reversing forward ones gives a measurable planner speedup.
- Timezone monotonicity is checked once when a zone loads, so date(timestamptz) stays usable in most zones.

**Thread state.** Zsolt Parragi reviewed repeatedly and found many wrong-result cases (time wraparound, interval arithmetic, NaN and infinity, domains, casts, DST, custom opclasses, cached plans), each answered by a fix; Corey Huinker and Dean Rasheed commented early. Since v16.0 in September only the author has posted, asking about a Windows-only test_plan_advice failure.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-11; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Demand is shown for time-bucket grouping only, but the scope is much wider

fixable · level 1b, rubric item 4 · demand, planner behaviour change, maintenance burden ("who maintains this in 10 years"), catalog changes · `demand-1`

A committer takes on prosupport entries for multiply, divide, unary minus, atan, sinh, erf, casts and timezone conversions. Users who never group by f(x) pay the extra planning cost. Each later fix for a wrong-result corner case, like the infinity, domain, cast or DST cases, lands on that committer.

The only use case anyone stated is group by date_trunc or ts::date (messages #1, #3). In message #12 the author says the ORDER BY case is "a bit pointless" because order by x works, and that min(f(x)) can be written as f(min(x)). Most wrong-result reports in messages #7, #11, #23, #26 and #33 came from arithmetic, interval, cast and infinity coverage that no user asked for. The author offered to limit the scope to GROUP BY in message #12, but v16 keeps the full scope.

Thread: [#1 (Alexandre Felipe, 2026-02-11)](https://www.postgresql.org/message-id/CAE8JnxN8kLUMygXb33LC-UPGyTC0cTcCyeWUdfEoz%3DOBo3aT5g%40mail.gmail.com), [#3 (Alexandre Felipe, 2026-03-20)](https://www.postgresql.org/message-id/CAE8JnxN_XzS1DgmEvEizbg7jXPgUwJ853uorciA5X8p%2B%3D9dYvA%40mail.gmail.com), [#12 (Alexandre Felipe, 2026-03-26)](https://www.postgresql.org/message-id/CAE8JnxP__hhKjP8uXKqGepg%2BGAjRy%2BJthg%3Df-e24G3PML7ZdbA%40mail.gmail.com), [#23 (Zsolt Parragi, 2026-05-08)](https://www.postgresql.org/message-id/CAN4CZFP%2BQkhGnt60_Z%2BQoaPj1nVfNtGFQhLtFxp_bTJ6SbMOMQ%40mail.gmail.com), [#26 (Zsolt Parragi, 2026-07-01)](https://www.postgresql.org/message-id/CAN4CZFPhBcNP6jxA4u%3DV%3Dmie%3DDNyYe4t8cENCzXk49ojkZ%3D5JQ%40mail.gmail.com), [#33 (Zsolt Parragi, 2026-07-21)](https://www.postgresql.org/message-id/CAN4CZFNDVZm9wj86APOTfrx2aJrd6TEJfqiceEe0XMO%3Da81wEw%40mail.gmail.com)

**Question:** Which users need ordering through arithmetic, trigonometric or cast expressions, and why would a version limited to date_trunc and date casts under GROUP BY not serve them?

Counter-check: held. The thread states one use case, grouping by date_trunc or a date cast. In message #8 the author says the other functions are there for completeness, and v16 keeps the full scope despite the offer in message #12 to narrow it.

### 2. Planning cost on every query for cases the author calls marginal

fixable · level 1b, rubric item 8 · performance on the hot path, planner behaviour change, new GUC · `maintenance-fit-3`

An OLTP workload with many indexes and ORDER BY on plain columns or casts pays extra planning time on every query. enable_slope is on by default, so these users pay a cost they never chose. The only numbers are from v5.

Message #10 shows about 5% overhead for ORDER BY a and 15% for casts on a table with 24 indexes, and v16 has much more logic than v5. In message #12 the author calls the ORDER BY and min/max cases "a bit pointless", because a rewrite gets the same plan, and offers to limit the analysis to GROUP BY. The full scope stayed (message #15), so all queries carry the cost of the broadest version.

Thread: [#10 (Alexandre Felipe, 2026-03-26)](https://www.postgresql.org/message-id/CAE8JnxOiw8B1FXb-fBGO-03xpVtZuZubCbpoW9LytmfFNB35Lw%40mail.gmail.com), [#12 (Alexandre Felipe, 2026-03-26)](https://www.postgresql.org/message-id/CAE8JnxP__hhKjP8uXKqGepg%2BGAjRy%2BJthg%3Df-e24G3PML7ZdbA%40mail.gmail.com), [#15 (Alexandre Felipe, 2026-04-05)](https://www.postgresql.org/message-id/CAE8JnxP75xOK9K7F0c74C9j4wnGEz9%2BxsuvwehExb3Lth3%2B3-A%40mail.gmail.com)

**Question:** What are the v16 planning-time numbers for queries that gain nothing, and would a GROUP BY-only scope keep most of the benefit for less cost?

Counter-check: held. The only planning-time numbers are from v5, and they show 5% to 15% overhead for queries that gain nothing. v6 restructured the analysis to cut per-index work, but no numbers for v16 are shown.

### 3. No measured gain for the motivating group by query

fixable · level 1b, rubric item 3 · demand, planner behaviour change · `demand-2`

A dashboard runs select date_trunc('month', ts), count(*) over a whole busy table. The planner now chooses a full index-only scan plus GroupAggregate, where today it would use a seq scan plus HashAggregate with a few groups. No run shows which plan is faster.

The thread shows EXPLAIN output and planning-time overhead from v5 (message #10), but no execution times before and after. For a full-table aggregate with few groups, today's hash plan may already be cheap, so the benefit likely depends on LIMIT, range filters or many groups. No one in the thread reports a field case or an independent user who hit this problem. Corey's "That's a good one" (message #4) is interest, not a report of need.

Thread: [#3 (Alexandre Felipe, 2026-03-20)](https://www.postgresql.org/message-id/CAE8JnxN_XzS1DgmEvEizbg7jXPgUwJ853uorciA5X8p%2B%3D9dYvA%40mail.gmail.com), [#4 (Corey Huinker, 2026-03-25)](https://www.postgresql.org/message-id/CADkLM%3Dc6-UuLNpN_O6e0Cuv7RWg1jS0bOd5WK_AJuPzR7JAD3Q%40mail.gmail.com), [#10 (Alexandre Felipe, 2026-03-26)](https://www.postgresql.org/message-id/CAE8JnxOiw8B1FXb-fBGO-03xpVtZuZubCbpoW9LytmfFNB35Lw%40mail.gmail.com)

**Question:** For which query shapes and table sizes does the new plan beat today's seq scan plus HashAggregate, and by how much in execution time?

Counter-check: held. The thread has EXPLAIN output and v5 planning times but no execution times and no independent user report. The scenario is weaker than stated, because the planner still costs the hash plan and only gains one more path.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Patches 0003 and 0007 add seven support functions and attach prosupport entries to many pg_proc rows. |
| ABI / extension API | **yes** | The patch adds SupportRequestMonotonic and changes the DecodeTimezoneName signature. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Patch 0004 adds enable_slope, default on. |
| planner behaviour change | **yes** | build_index_pathkeys emits extra pathkeys, so plans for ORDER BY, GROUP BY, DISTINCT and min/max change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each monotonicity annotation is a correctness promise, and the patches couple the planner to src/timezone and session TimeZone. |
| performance on the hot path | **yes** | Slope analysis runs at plan time for every query; v5 numbers show 1% to 15% planning overhead with 24 indexes. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What are the v16 planning-time numbers for queries that gain nothing from slope analysis, on tables with many indexes?
2. For which query shapes and table sizes does the new plan beat today's seq scan plus HashAggregate in execution time, and by how much?
3. Which users need ordering through arithmetic, trigonometric or cast expressions, and would a version limited to date_trunc and date casts under GROUP BY serve the stated use case?
4. Which plan-caching paths (PL/pgSQL, SPI, extended protocol, force_generic_plan) does the guard cover, and what invalidates a cached plan when TimeZone changes?
5. What mechanism, beyond regression tests for known cases, would catch a wrong or stale monotonicity annotation before it ships?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pathkeys and equivalence classes in the planner, and one who knows the datetime and timezone code. Dean Rasheed commented early and asked to defer; no such committer has reviewed the design since.
- **Liability:** The owner signs for the correctness of every monotonicity annotation in pg_proc, for the plan-cache guard, and for the planner's dependence on timezone data. A wrong annotation gives wrongly ordered or grouped rows with no error, for as long as the feature exists.
- **Author follow-up:** Good sign: the author answered each reported wrong-result case with a fix through 16 versions and was still posting after v16.0 in September.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Grouping by time bucket is a common report pattern, but v16 is eight patches across the planner, pg_proc, datetime and src/timezone, and its scope exceeds the stated demand. A narrower version with v16 measurements would be cheaper to review and own; patches 0002 and 0005 look separable and could go first.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Cached plans can return wrongly ordered rows after a session setting changes. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) The SupportRequestMonotonic contract does not name its ordering. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Correctness rests on hand-kept pg_proc annotations that nothing checks. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-3` enable_slope works as a wrong-results escape hatch. Refuted (no failure scenario): The scenario assumes a wrong-result case nobody has found yet, which is the concern in maintenance-fit-1. The setting follows the existing enable_* pattern, a reviewer asked for it in message #28, and comparing results with it off is an ordinary test method.
- `maintenance-fit-2` Planner correctness now depends on timezone data and session state. Refuted (not specific): Duplicate of standards-compat-1 for the cached-plan scenario. The tzdata part has no concrete failure: Postgres already ships and depends on tzdata, and plans do not outlive the session that loaded the zone.
- `already-exists-1` ORDER BY f(x) and min(f(x)) already have simple rewrites. Refuted (not specific): Duplicate of demand-1, which makes the scope point with better support. Message #12 also says ORDER BY support comes from the same pathkey mechanism that GROUP BY needs, so it adds no separate wrong-result risk.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] is about using expression indexes for ORDER BY, which is a different direction from this proposal's use of a plain index on x for ORDER BY or GROUP BY f(x). [P1] also records no reason, so nothing from it can be applied here. No thread message points to an earlier discussion of this idea; message #40 links only a separate date_trunc bug patch.
- standards-compat: 3 objection(s). There are objections on the standards and compatibility angle. pg_dump round-trip and SQL syntax are not affected, because the patch adds no syntax and changes only built-in pg_proc entries.
- replication-ondisk: 0 objection(s). The proposal adds pg_proc prosupport entries, a new support request, a planner-only cache and a GUC. It does not change on-disk format, WAL or the wire protocol. pg_upgrade recreates built-in pg_proc rows, and a standby plans with its own tzdata and session settings. Extensions that call DecodeTimezoneName need a recompile, which is normal for a major release. This angle has no replication or on-disk objection.
- security: 0 objection(s). The proposal changes how the planner uses existing index order and adds no new privilege or data exposure. Only superusers can attach a SUPPORT function or create an operator class, so unprivileged users cannot declare false monotonicity. The planning-time calls to timezone loading and equalimage functions use catalog or constant input, and SQL execution can already reach both. Wrong-result risks are real, but they are correctness issues for the other angles, not escalation or leak paths. Without the code, I could not check that the time-zone scan at zone load has bounded cost.
- maintenance-fit: 3 objection(s). Patches 0002 (reverse pathkeys) and 0005 (constant sort keys) look separable and could go in first. No committer who knows this planner area has reviewed the design since Dean Rasheed asked to defer it (message #13).
- already-exists: 1 objection(s). There is one objection. GROUP BY date_trunc on several grains with a single index has no exact equivalent today, so that core use case is not disputed from this angle.
- demand: 2 objection(s). Two objections raised. Both concern scope and missing evidence of gain, not whether time-bucket grouping is a real pattern.

### Precedents retrieved

- P1: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - used in the review
- P2: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P3: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P4: [Problem in 'ORDER BY' of a column using a created collation?](https://commitfest.postgresql.org/patch/5990/) (cf, Returned with feedback, 2025-10-21) - judged not relevant; reason summarised from its thread
- P5: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P6: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant; reason summarised from its thread
- P7: [logical column ordering (WIP)](https://commitfest.postgresql.org/patch/21/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P8: [Optimize planner memory consumption for huge arrays](https://commitfest.postgresql.org/patch/4825/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [PROXY protocol support](https://commitfest.postgresql.org/patch/3032/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P11: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant; reason summarised from its thread
- P12: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P13: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant
- P14: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P15: [GnuTLS support](https://commitfest.postgresql.org/patch/1277/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P16: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P17: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P18: [Introduce the ability to enable/disable indexes](https://commitfest.postgresql.org/patch/5267/) (cf, Withdrawn, 2024-09-26) - judged not relevant
- P19: [BRIN Sort - sorting using BRIN indexes](https://commitfest.postgresql.org/patch/3949/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P20: [seccomp-bpf support](https://commitfest.postgresql.org/patch/2263/) (cf, Rejected, 2020-01-07) - judged not relevant
- P21: [Optimizing Node Files Support](https://commitfest.postgresql.org/patch/4034/) (cf, Rejected, 2023-03-21) - judged not relevant
- P22: [support for MODULE](https://commitfest.postgresql.org/patch/3528/) (cf, Returned with feedback, 2022-07-18) - judged not relevant
- P23: [Online enabling of checksums](https://commitfest.postgresql.org/patch/1535/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P24: [Revert 1fbc30f1dca6624c01a9a6a9ff711a7036c68532 because the --enable-shared is about Plyton's configure, not ours.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39d0653d058b87b437be05fd295ceb5bb4e9e05c) (revert, Reverted, 2012-08-30) - judged not relevant

### Warnings

- attachment limit (8) reached; v16.0-0009-SLOPE-plan-cache-guards.patch not read
- attachment limit (8) reached; 0002-SLOPE-Builtin-support.patch not read
- attachment limit (8) reached; 0003-SLOPE-Tests.patch not read
- attachment limit (8) reached; 0001-SLOPE-Analysis-Machinery.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 175 s; cost reported by the CLI: $4.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 96 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:52:45.412Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
