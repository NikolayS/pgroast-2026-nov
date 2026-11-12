# Design pre-review: Add get_actual_variable_range index page read limit (complements existing VISITED_PAGES_LIMIT)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7305](https://commitfest.postgresql.org/patch/7305/); [thread](https://www.postgresql.org/message-id/flat/CAH2-Wzkt1WkKp4VRJu3qHfmKXc8W%2BXYv1RXg5d2d3fSvAeO%3Drg%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is real and agreed: Tom Lane confirmed it in message #2, and the demand angle found a production case. The one surviving objection, prior-attempts-1, is a fixable design point. INDEX_PAGES_LIMIT = 3 can end the probe where master finds the true endpoint, and no posted numbers measure how often. The objectors to the earlier design have not replied to the revised one, so a committer who knows the planner must judge that trade-off.

Not checked: No code was read. Whether the counter covers all nbtree scan paths (backward scans, parallel scans, array keys) and adds no cost for ordinary scans could not be checked.

## Prior attempts

None found before 2026-02-09.

## Objections a reviewer is likely to raise

### 1. Revised limit still gives up early, and no failure-rate data answers message #2

A batch insert at the high end of a timestamp index rolls back on a unique conflict. Later planning calls kill those entries and find more than three leaf pages with no live items. The planner falls back to a stale pg_statistic maximum. Range queries on recent rows get underestimated, and the DBA sees a bad plan with no sign of the cause.

In message #2, Tom Lane rejected the single-page design for its higher chance of getting no estimate, and suggested a limit of about a thousand dead entries. Message #7 gave the rollback case above. INDEX_PAGES_LIMIT counts empty pages, so three sparse pages holding a few dead entries trigger it as easily as three full ones. Message #8 argues the case is short-lived, but neither objector replied to v1, and all posted numbers are for the earlier design.

**Question:** With INDEX_PAGES_LIMIT = 3, how often does get_actual_variable_range return no estimate in the rollback and queue cases where master finds the true endpoint?

<sub>fixable, level 2, planner behaviour change; messages [#2](https://www.postgresql.org/message-id/3613043.1770683973%40sss.pgh.pa.us), [#7](https://www.postgresql.org/message-id/4s3vd2rjll4vgnhdnajrqwsnhnmnmtmjtwjngfsrzmjx3nrxvp%40la2vohpjojnk), [#8](https://www.postgresql.org/message-id/CAH2-WzkmWviX6fGwJ_a775WMQ5-Qvg_FPXg%3D5wA4DMDeScmURw%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAH2-WzmugSOCGw-%2BmjocGRXBbwSevjMJ41dLaF0ukJzUGEKF%3DQ%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAH2-Wz%3DJ2d3TFJ1QLTia5Hr-kJXk%2BURBT8h10vGi6hO82L3kjw%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. With INDEX_PAGES_LIMIT = 3, how often does get_actual_variable_range return no estimate in the rollback and queue cases where master finds the true endpoint?
2. What are the planning latency numbers for the v1 design, since all posted numbers are for the earlier single-leaf-page design?
3. How was the value 3 chosen, and how do failure rate and latency change with a larger value?
4. Why does the limit count empty leaf pages instead of dead index entries skipped, as message #2 suggested with a threshold of about a thousand?
5. Is the fix meant for back branches, and if so, how does the relscan.h field change stay ABI-safe there?

## Touch points

- **ABI / extension API:** The patch adds a scan descriptor field in relscan.h that only nbtree honors.
- **planner behaviour change:** get_actual_variable_range falls back to pg_statistic after three empty leaf pages, which changes row estimates for existing queries.
- **backpatchability:** The proposal does not say whether back branches get the fix, and the relscan.h struct change would need an ABI-safe form there.
- **maintenance burden ("who maintains this in 10 years"):** A second hard-coded limit, counted only in _bt_readnextpage, must survive later reworks of nbtree scans.
- **performance on the hot path:** The patch adds counting in _bt_readnextpage, which every nbtree scan that moves to a next leaf page runs.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Geoghegan wrote the patch and works in nbtree; Tom Lane took part in message #2. Someone who knows selfuncs.c and planner estimation should agree to the estimate-loss trade-off.
- Priority: proceed; audience a sizeable minority; review cost low. Queue-like tables are common, and the thread shows a published benchmark and a production report. The patch is small, adds no SQL or GUC surface, and needs failure-rate numbers for the revised design before commit.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
