# Design pre-review: Randomize B-Tree page split location to avoid oscillating patterns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6892](https://commitfest.postgresql.org/patch/6892/); [thread](https://www.postgresql.org/message-id/flat/d6do2mtjcsagn37jf6pjywzhlzyokqja6jlnvcs4ypkvnnuu32%40llwuuyxxomc3), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The design has the reviewers' agreement for SPLIT_DEFAULT leaf splits, but the benefit is not yet shown (demand-1). Message #12 sets the bar at lower tail latency with no loss of page occupancy, and v3 in message #14 brings no such result. The cost of the full scan is measured on one single-column index only (demand-2). This is a "not now, show the numbers" outcome, not a rejection.

Not checked: Without the code, we could not check whether v3 scans the whole split interval, which random source it uses, or whether the new test is stable. The repeatability objection (standards-compat-1) is at level 2 and was not evaluated here.

## Prior attempts

None found before 2026-04-27.

## Objections a reviewer is likely to raise

### 1. No field report shows users hurt by split oscillation

A committer pushes the change based on graphs of split counts from RAM-disk and unlogged-table runs. Users on real storage see no change in latency, and some indexes end up slightly larger. Nobody can show that the extra nbtree policy fixed a problem anyone reported.

The evidence of demand is one paper and the author's synthetic tests. Message #1 says the effect on performance could not be measured, and message #4 says latency showed nothing beyond noise. Message #2 calls the phenomenon real but is unsure of its importance. Message #12 sets the bar at a statistically significant tail-latency gain with no significant loss of occupancy, and no v3 result in message #14 meets it.

**Question:** On logged tables and real storage, does v3 cut p99 or worse insert latency for a UUID or similar workload, and is average leaf occupancy unchanged against master?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/d6do2mtjcsagn37jf6pjywzhlzyokqja6jlnvcs4ypkvnnuu32%40llwuuyxxomc3), [#2](https://www.postgresql.org/message-id/CAH2-Wz%3DSy7%3DkAHrtNrDSoo%2Bvj_QYAmNc8_GEYPh6r%2B0r-BbQ5w%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/uqcsdatuokodytg7dcan75hbutv4erwdmzqaxjwhakmvfhy7vf%40fxlpamb3opzr), [#7](https://www.postgresql.org/message-id/jotfal6422zpqvux4add7xcyftaujjiksscfcc5uuibaldw4sv%40pzriktpx4gyl), [#12](https://www.postgresql.org/message-id/0025135A-F56E-44FF-95C8-945A6BB5FDB4%40yandex-team.ru), [#14](https://www.postgresql.org/message-id/atCTpUNwWNm3k2Lg%40ddolgov-thinkpadt14sgen1.rmtde.csb); `demand-1`</sub>

### 2. Every default leaf split pays for a gain few workloads see

An insert-heavy workload on a unique index never oscillates in practice. After the change, each of its leaf splits scans the whole split interval and draws random numbers, where before the search stopped after one step. The workload pays the cost and gets nothing back.

Message #6 notes that unique indexes now finish the search after one iteration, and the full scan from message #8 removes that early exit for all SPLIT_DEFAULT splits. The benefit applies only to workloads whose new keys follow the existing distribution, and message #4 says suffix truncation already spreads splits on wider indexes. Message #12 names the added nbtree complexity as the main cost. The only cost data so far is one single-column test in message #14.

**Question:** What does the full-interval scan cost on multi-column and non-unique indexes, and which common workloads gain enough to justify it?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/uqcsdatuokodytg7dcan75hbutv4erwdmzqaxjwhakmvfhy7vf%40fxlpamb3opzr), [#6](https://www.postgresql.org/message-id/noovhlxtwa64do7isbbvl2f6clqukujb7hexw3zi2rn3ssbmrw%403yvb4ghojg4b), [#8](https://www.postgresql.org/message-id/FB98633D-4D3F-4FB5-9CEA-240ACBDAE4F1%40yandex-team.ru), [#12](https://www.postgresql.org/message-id/0025135A-F56E-44FF-95C8-945A6BB5FDB4%40yandex-team.ru), [#14](https://www.postgresql.org/message-id/atCTpUNwWNm3k2Lg%40ddolgov-thinkpadt14sgen1.rmtde.csb); `demand-2`</sub>

## Questions for the author

1. On logged tables and real storage, does v3 lower p99 or worst-case insert latency against master for a UUID index workload?
2. Is average leaf page occupancy, or final index size, the same with v3 as on master for the same workloads?
3. What does the full scan of the split interval cost per split on multi-column and non-unique indexes?
4. Which random number source does v3 use, and how does a regression test or a developer get a repeatable leaf layout from the same input?
5. Does v3 scan the whole split interval, as the review asked, or search 20% of the locations, as the commit message of 0003 says?

## Touch points

- **backpatchability:** The CommitFest entry gives target version "stable", but the patch changes split behaviour and is not a bug fix.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a random split policy to nbtsplitloc.c, four USDT probes in probes.d, and a test that depends on random outcomes.
- **performance on the hot path:** Every SPLIT_DEFAULT leaf split scans the whole split interval, where a unique index stopped after one step before.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No field report shows users hurt by split oscillation

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Geoghegan, who knows the nbtree split location code, reviews the patch in the thread and agrees with the approach for SPLIT_DEFAULT leaf splits.
- Not now, show demand first: A statistically significant drop in tail insert latency on logged tables and real storage, with average leaf occupancy unchanged against master. Also the cost of the full scan on multi-column and non-unique indexes.
- Priority: proceed; audience unclear; review cost medium. The patch is small and limited to nbtsplitloc.c, and a committer who knows the area is engaged. The thread has no measurement that shows which users feel the oscillation, so the next step is the benchmark the reviewers asked for.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
