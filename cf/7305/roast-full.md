# Design pre-review: Add get_actual_variable_range index page read limit (complements existing VISITED_PAGES_LIMIT)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7305](https://commitfest.postgresql.org/patch/7305/) (Needs review); authors: Peter Geoghegan (pgeoghegan)
- Thread: [Problems with get_actual_variable_range's VISITED_PAGES_LIMIT](https://www.postgresql.org/message-id/flat/CAH2-Wzkt1WkKp4VRJu3qHfmKXc8W%2BXYv1RXg5d2d3fSvAeO%3Drg%40mail.gmail.com), started 2026-02-09
- Messages read: 12 of 12
- Patch prose read: v1-0001-Limit-get_actual_variable_range-leaf-page-reads.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is real and agreed: Tom Lane confirmed it in message #2, and the demand angle found a production case. The one surviving objection, prior-attempts-1, is a fixable design point. INDEX_PAGES_LIMIT = 3 can end the probe where master finds the true endpoint, and no posted numbers measure how often. The objectors to the earlier design have not replied to the revised one, so a committer who knows the planner must judge that trade-off.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read. Whether the counter covers all nbtree scan paths (backward scans, parallel scans, array keys) and adds no cost for ordinary scans could not be checked.

## The proposal, restated

**Problem.** The planner's get_actual_variable_range can read an unbounded number of index leaf pages when many index tuples are marked LP_DEAD, because VISITED_PAGES_LIMIT counts only heap page visits. Queue-like tables, such as those in Mark Callaghan's insert benchmark and possibly TPC-C's order table, suffer high planning latency as a result.

**Proposed design.** The patch adds INDEX_PAGES_LIMIT, set to 3, which counts leaf pages that return zero matching items. The scan gives up when the count hits the limit, and the planner falls back to pg_statistic. VISITED_PAGES_LIMIT stays as is, and only nbtree's _bt_readnextpage counts pages, so the first leaf page read is never counted.

**User-visible surface.**
- C API: new scan descriptor field in relscan.h, named xs_index_pages_limit in an earlier version, that callers set to ask the index AM to stop early.
- No SQL, GUC, catalog or protocol change.

**What the author claims.**
- For the earlier single-leaf-page design, EXPLAIN latency with 20% deleted tuples was about 10 times lower than master, with no change when there are no dead tuples.
- With the new limit, standbys keep today's behavior, because LP_DEAD bits are not set or used there and each page usually returns a match.
- Counting only in _bt_readnextpage adds no overhead to simple point queries.
- Reading more than a few empty leaf pages at the end of the index indicates a pathological case, so reading more pages is unlikely to find a value.
- The patch no longer depends on the index prefetching work, because the slot-based table AM index scan interface is now committed.

**Thread state.** Tom Lane objected to the first single-page design and Andres Freund doubted index cost as a proxy for heap cost; the author then changed to the complementary limit. Neither has replied to the revised design, and Rui Zhao is listed as reviewer with no review posted.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Revised limit still gives up early, and no failure-rate data answers message #2

fixable · level 2, rubric item 4 · planner behaviour change · `prior-attempts-1`

A batch insert at the high end of a timestamp index rolls back on a unique conflict. Later planning calls kill those entries and find more than three leaf pages with no live items. The planner falls back to a stale pg_statistic maximum. Range queries on recent rows get underestimated, and the DBA sees a bad plan with no sign of the cause.

In message #2, Tom Lane rejected the single-page design for its higher chance of getting no estimate, and suggested a limit of about a thousand dead entries. Message #7 gave the rollback case above. INDEX_PAGES_LIMIT counts empty pages, so three sparse pages holding a few dead entries trigger it as easily as three full ones. Message #8 argues the case is short-lived, but neither objector replied to v1, and all posted numbers are for the earlier design.

Thread: [#2 (Tom Lane, 2026-02-10)](https://www.postgresql.org/message-id/3613043.1770683973%40sss.pgh.pa.us), [#7 (Andres Freund, 2026-03-10)](https://www.postgresql.org/message-id/4s3vd2rjll4vgnhdnajrqwsnhnmnmtmjtwjngfsrzmjx3nrxvp%40la2vohpjojnk), [#8 (Peter Geoghegan, 2026-03-13)](https://www.postgresql.org/message-id/CAH2-WzkmWviX6fGwJ_a775WMQ5-Qvg_FPXg%3D5wA4DMDeScmURw%40mail.gmail.com), [#9 (Peter Geoghegan, 2026-06-23)](https://www.postgresql.org/message-id/CAH2-WzmugSOCGw-%2BmjocGRXBbwSevjMJ41dLaF0ukJzUGEKF%3DQ%40mail.gmail.com), [#12 (Peter Geoghegan, 2026-09-15)](https://www.postgresql.org/message-id/CAH2-Wz%3DJ2d3TFJ1QLTia5Hr-kJXk%2BURBT8h10vGi6hO82L3kjw%40mail.gmail.com)

**Question:** With INDEX_PAGES_LIMIT = 3, how often does get_actual_variable_range return no estimate in the rollback and queue cases where master finds the true endpoint?

Counter-check: held. The limit counts leaf pages with zero matching items, so a rolled-back batch spanning a few leaf pages can end the probe where master finds the endpoint. Message #8's answer covers only workloads that retry, and no posted numbers measure the revised design.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a scan descriptor field in relscan.h that only nbtree honors. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | get_actual_variable_range falls back to pg_statistic after three empty leaf pages, which changes row estimates for existing queries. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The proposal does not say whether back branches get the fix, and the relscan.h struct change would need an ABI-safe form there. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second hard-coded limit, counted only in _bt_readnextpage, must survive later reworks of nbtree scans. |
| performance on the hot path | **yes** | The patch adds counting in _bt_readnextpage, which every nbtree scan that moves to a next leaf page runs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With INDEX_PAGES_LIMIT = 3, how often does get_actual_variable_range return no estimate in the rollback and queue cases where master finds the true endpoint?
2. What are the planning latency numbers for the v1 design, since all posted numbers are for the earlier single-leaf-page design?
3. How was the value 3 chosen, and how do failure rate and latency change with a larger value?
4. Why does the limit count empty leaf pages instead of dead index entries skipped, as message #2 suggested with a threshold of about a thousand?
5. Is the fix meant for back branches, and if so, how does the relscan.h field change stay ABI-safe there?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Geoghegan wrote the patch and works in nbtree; Tom Lane took part in message #2. Someone who knows selfuncs.c and planner estimation should agree to the estimate-loss trade-off.
- **Liability:** The owner signs for plan regressions when the probe gives up early and the planner uses a stale pg_statistic endpoint. That lasts as long as the hard-coded limit exists.
- **Author follow-up:** The author changed the design after the objections in the thread and posted v1 in message #12, which suggests continued work.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Queue-like tables are common, and the thread shows a published benchmark and a production report. The patch is small, adds no SQL or GUC surface, and needs failure-rate numbers for the revised design before commit.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Existing range queries can lose actual-endpoint estimates with no way back. Refuted (not specific): Duplicate of prior-attempts-1, which states the same rollback scenario and the same missing failure-rate data. The added points about a setting and back branches name no separate failure.
- `maintenance-fit-1` Scan-descriptor limit has an nbtree-only contract that can drop silently. Refuted (no failure scenario): An index AM or scan path that ignores the field behaves as master does today, so no new failure arises, and the patch comment shows the committed xs_visited_pages_limit follows the same pattern. The patch was carried inside the prefetching series through v28, by the same author.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the index. The thread names commit 9c6ad5eaa9, which added VISITED_PAGES_LIMIT, as background (messages #1 and #4). Message #4 calls it a quick backpatchable fix, which this patch extends rather than reverses.
- standards-compat: 1 objection(s). The proposal adds no SQL, GUC, catalog, dump or psql surface, so the only compatibility concern from this angle is the planner behaviour change for existing queries.
- replication-ondisk: 0 objection(s). The patch changes no on-disk format, WAL record, catalog or wire protocol. It adds only a scan descriptor field in relscan.h, which is a normal change on master. On a hot standby, LP_DEAD bits are not used, so as message #9 says the new limit rarely triggers there and behaviour stays as today. The nbtree VACUUM bookkeeping invariant does not apply, because the patch does not change VACUUM. If the fix is later proposed for back branches, the relscan.h struct change would need an ABI-safe form, but the proposal does not ask for a backpatch.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds an internal limit on leaf page reads during planner endpoint probes, with no new SQL, GUC, catalog, role or protocol surface, and it lowers rather than raises the work an unprivileged user can trigger by planning a query.
- maintenance-fit: 1 objection(s). From the maintenance angle the cost looks small: there is no GUC, catalog or SQL surface, and the extra work sits in _bt_readnextpage only. The nbtree VACUUM invariant does not apply because this is a planner-time scan. The author is the committer who owns the area.
- already-exists: 0 objection(s). I found no existing Postgres feature, setting or extension that bounds leaf page reads in get_actual_variable_range, and the limit lives inside planner and nbtree code that no hook reaches. Aggressive vacuuming of queue tables only reduces how often the problem occurs, and the new limit extends VISITED_PAGES_LIMIT instead of adding a second way to do the same job.
- demand: 0 objection(s). The material shows real demand: Mark Callaghan's published benchmark, Tom Lane agreeing the problem exists (message #2), and Jakub Wartak's production case where planning took about 27 minutes (message #4). The patch has no user-visible surface, and the open questions are about how often estimates fail, which other angles cover.

### Precedents retrieved

- P1: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant; reason summarised from its thread
- P2: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P3: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P4: [Revert "Get rid of the "new" and "old" entries in a view's rangetable."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f0e6d6d3c909e14eedd370dd77d7111c2ae24c02) (revert, Reverted, 2023-01-12) - judged not relevant
- P5: [Remove dead macro exec_subplan_get_plan](https://commitfest.postgresql.org/patch/3894/) (cf, Rejected, 2023-04-04) - judged not relevant; reason summarised from its thread
- P6: [Full support for index LP_DEAD hint bits on standby](https://commitfest.postgresql.org/patch/2947/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P7: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P8: [Revert change to autodetect '__getopt_initialized' variable ... turns out glibc doesn't actually export this variable, so we need a different solution.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2b7206a9935c915155a2c71e4658638de98ad440) (revert, Reverted, 2001-10-20) - judged not relevant
- P9: [Adjust tuples estimate for appendrel](https://commitfest.postgresql.org/patch/4711/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P10: [Revert "Disallow infinite endpoints in generate_series() for timestamps."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29904f5f2fdafbbb96ef3685fd361053b061aeb1) (revert, Reverted, 2022-05-09) - judged not relevant
- P11: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant; reason summarised from its thread
- P12: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P13: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P14: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P15: [Suggesting an improvement for indexonlyscan cost](https://commitfest.postgresql.org/patch/2759/) (cf, Withdrawn, 2020-10-13) - judged not relevant
- P16: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P18: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P19: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Bloom index cost model](https://commitfest.postgresql.org/patch/2043/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P22: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P23: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P24: [Avoiding plan disasters with LIMIT](https://commitfest.postgresql.org/patch/29/) (cf, Returned with feedback, 2015-05-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 122 s; cost reported by the CLI: $1.59
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:38:33.375Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
