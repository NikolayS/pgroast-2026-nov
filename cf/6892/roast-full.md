# Design pre-review: Randomize B-Tree page split location to avoid oscillating patterns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6892](https://commitfest.postgresql.org/patch/6892/) (Needs review); authors: Dmitry Dolgov (erthalion)
- Thread: [Randomize B-Tree page split location to avoid oscillating patterns](https://www.postgresql.org/message-id/flat/d6do2mtjcsagn37jf6pjywzhlzyokqja6jlnvcs4ypkvnnuu32%40llwuuyxxomc3), started 2026-04-27
- Messages read: 14 of 14
- Patch prose read: v3-0001-Add-USDT-for-nbtree-page-splits-and-suffix-trunca.patch, v3-0002-Add-USDT-for-split-location-benchmarking.patch, v3-0003-Randomize-nbtree-split-location-to-avoid-oscillat.patch, v1-0001-Add-a-USDT-for-tracing-nbtree-page-splits.patch, v1-0002-Randomize-nbtree-split-location-to-avoid-oscillat.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The design has the reviewers' agreement for SPLIT_DEFAULT leaf splits, but the benefit is not yet shown (demand-1). Message #12 sets the bar at lower tail latency with no loss of page occupancy, and v3 in message #14 brings no such result. The cost of the full scan is measured on one single-column index only (demand-2). This is a "not now, show the numbers" outcome, not a rejection.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, we could not check whether v3 scans the whole split interval, which random source it uses, or whether the new test is stable. The repeatability objection (standards-compat-1) is at level 2 and was not evaluated here.

## The proposal, restated

**Problem.** Under some insert workloads nbtree picks the same split location again and again, so many leaf pages fill and split at about the same time. The author says the number of page splits then oscillates over time (the "waves of misery" effect), which varies IO and buffer contention. UUID indexes and new data that matches the existing key distribution are named as bad cases.

**Proposed design.** In nbtsplitloc.c, the split finder scans the whole existing split interval and picks at random among the locations with the true minimum split penalty, using reservoir sampling. So suffix truncation keeps its current effect while the split location varies. Per the review, the scope is SPLIT_DEFAULT leaf splits only; SPLIT_MANY_DUPLICATES and internal pages do not change.

**User-visible surface.**
- USDT probe nbtree__page__split with a BlockNumber argument.
- USDT probe nbtree__page__truncate with the number of truncated columns.
- USDT probes nbtree__page__findsplit__start and nbtree__page__findsplit__done, which the author says are for benchmarking only.
- New regression test nbtree_split_randomization in src/test/modules/nbtree.

**What the author claims.**
- With randomization, the page split rate stops oscillating in a single-column unique integer index test and a UUID batch-load test.
- pgbench latency stddev improved in the single-column unique index test, run with more throughput than the first tests.
- Reservoir sampling was only slightly faster than keeping an array of candidates, and was chosen because it fits the code better.
- The 20% figure comes from the Glombiewski, Seeger and Graefe paper.
- The author could not show a clear throughput change; tests ran on a RAM disk or unlogged tables.

**Thread state.** Peter Geoghegan and Andrey Borodin (reviewer) agree with the approach when limited to SPLIT_DEFAULT leaf splits, and v3 follows that. They still want evidence that tail latency drops while average page occupancy does not; Borodin's separate batch extension patch is a side topic.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No field report shows users hurt by split oscillation

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer pushes the change based on graphs of split counts from RAM-disk and unlogged-table runs. Users on real storage see no change in latency, and some indexes end up slightly larger. Nobody can show that the extra nbtree policy fixed a problem anyone reported.

The evidence of demand is one paper and the author's synthetic tests. Message #1 says the effect on performance could not be measured, and message #4 says latency showed nothing beyond noise. Message #2 calls the phenomenon real but is unsure of its importance. Message #12 sets the bar at a statistically significant tail-latency gain with no significant loss of occupancy, and no v3 result in message #14 meets it.

Thread: [#1 (Dmitry Dolgov, 2026-04-27)](https://www.postgresql.org/message-id/d6do2mtjcsagn37jf6pjywzhlzyokqja6jlnvcs4ypkvnnuu32%40llwuuyxxomc3), [#2 (Peter Geoghegan, 2026-04-27)](https://www.postgresql.org/message-id/CAH2-Wz%3DSy7%3DkAHrtNrDSoo%2Bvj_QYAmNc8_GEYPh6r%2B0r-BbQ5w%40mail.gmail.com), [#4 (Dmitry Dolgov, 2026-05-06)](https://www.postgresql.org/message-id/uqcsdatuokodytg7dcan75hbutv4erwdmzqaxjwhakmvfhy7vf%40fxlpamb3opzr), [#7 (Dmitry Dolgov, 2026-06-16)](https://www.postgresql.org/message-id/jotfal6422zpqvux4add7xcyftaujjiksscfcc5uuibaldw4sv%40pzriktpx4gyl), [#12 (Andrey Borodin, 2026-08-29)](https://www.postgresql.org/message-id/0025135A-F56E-44FF-95C8-945A6BB5FDB4%40yandex-team.ru), [#14 (Dmitry Dolgov, 2026-10-15)](https://www.postgresql.org/message-id/atCTpUNwWNm3k2Lg%40ddolgov-thinkpadt14sgen1.rmtde.csb)

**Question:** On logged tables and real storage, does v3 cut p99 or worse insert latency for a UUID or similar workload, and is average leaf occupancy unchanged against master?

Counter-check: held. Message #7 shows a pgbench latency stddev gain and both reviewers call the idea sound, so the objection understates the evidence. But messages #12 and #13 still ask for tail latency and occupancy results, and message #14 gives none.

### 2. Every default leaf split pays for a gain few workloads see

fixable · level 1b, rubric item 4 · demand, performance on the hot path, maintenance burden ("who maintains this in 10 years") · `demand-2`

An insert-heavy workload on a unique index never oscillates in practice. After the change, each of its leaf splits scans the whole split interval and draws random numbers, where before the search stopped after one step. The workload pays the cost and gets nothing back.

Message #6 notes that unique indexes now finish the search after one iteration, and the full scan from message #8 removes that early exit for all SPLIT_DEFAULT splits. The benefit applies only to workloads whose new keys follow the existing distribution, and message #4 says suffix truncation already spreads splits on wider indexes. Message #12 names the added nbtree complexity as the main cost. The only cost data so far is one single-column test in message #14.

Thread: [#4 (Dmitry Dolgov, 2026-05-06)](https://www.postgresql.org/message-id/uqcsdatuokodytg7dcan75hbutv4erwdmzqaxjwhakmvfhy7vf%40fxlpamb3opzr), [#6 (Dmitry Dolgov, 2026-05-07)](https://www.postgresql.org/message-id/noovhlxtwa64do7isbbvl2f6clqukujb7hexw3zi2rn3ssbmrw%403yvb4ghojg4b), [#8 (Andrey Borodin, 2026-08-24)](https://www.postgresql.org/message-id/FB98633D-4D3F-4FB5-9CEA-240ACBDAE4F1%40yandex-team.ru), [#12 (Andrey Borodin, 2026-08-29)](https://www.postgresql.org/message-id/0025135A-F56E-44FF-95C8-945A6BB5FDB4%40yandex-team.ru), [#14 (Dmitry Dolgov, 2026-10-15)](https://www.postgresql.org/message-id/atCTpUNwWNm3k2Lg%40ddolgov-thinkpadt14sgen1.rmtde.csb)

**Question:** What does the full-interval scan cost on multi-column and non-unique indexes, and which common workloads gain enough to justify it?

Counter-check: held. Messages #12 and #13 weigh the CPU cost as minor, and message #7 shows the split search takes tens of nanoseconds on a single-column index. Message #8 asked for the full-scan cost to be measured, and no multi-column or non-unique result exists yet.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry gives target version "stable", but the patch changes split behaviour and is not a bug fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a random split policy to nbtsplitloc.c, four USDT probes in probes.d, and a test that depends on random outcomes. |
| performance on the hot path | **yes** | Every SPLIT_DEFAULT leaf split scans the whole split interval, where a unique index stopped after one step before. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No field report shows users hurt by split oscillation |

## 5. Questions the author must answer before anyone reviews code

1. On logged tables and real storage, does v3 lower p99 or worst-case insert latency against master for a UUID index workload?
2. Is average leaf page occupancy, or final index size, the same with v3 as on master for the same workloads?
3. What does the full scan of the split interval cost per split on multi-column and non-unique indexes?
4. Which random number source does v3 use, and how does a regression test or a developer get a repeatable leaf layout from the same input?
5. Does v3 scan the whole split interval, as the review asked, or search 20% of the locations, as the commit message of 0003 says?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Geoghegan, who knows the nbtree split location code, reviews the patch in the thread and agrees with the approach for SPLIT_DEFAULT leaf splits.
- **Liability:** The owner signs for the split policy of every default leaf split in all nbtree indexes, with no time limit. This includes any later report of larger indexes and any layout that no longer reproduces from the same inserts.
- **Author follow-up:** The author posted three versions that follow the review and moved the entry through three CommitFests.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A statistically significant drop in tail insert latency on logged tables and real storage, with average leaf occupancy unchanged against master. Also the cost of the full scan on multi-column and non-unique indexes.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is small and limited to nbtsplitloc.c, and a committer who knows the area is engaged. The thread has no measurement that shows which users feel the oscillation, so the next step is the benchmark the reviewers asked for.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Split choice may not be repeatable for tests and investigations. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` Benchmark-only USDT probes would become a permanent tracing interface. Refuted (already handled): Message #7 says 0001 "makes sense to have in general" and 0002 is "purely for benchmarking", so the author presents 0002 as a measurement aid for the review. The missing documentation for the 0001 probes is a level 4 matter, out of scope here.
- `maintenance-fit-1` Benchmarking-only USDT probes would become permanent core surface. Refuted (already handled): The failure scenario needs 0002 to be committed, and message #7 labels 0002 "purely for benchmarking", apart from 0001. Randomization in 0003 does not depend on it, so a committer can leave 0002 out without any change to the proposal.
- `maintenance-fit-2` Random split choice makes nbtree layout non-reproducible. Refuted (not specific): Duplicate of standards-compat-1, which states the same random-source and test-stability question with a more concrete scenario.
- `already-exists-1` Benchmark-only findsplit probes duplicate what uprobes already give. Refuted (already handled): Message #7 already presents 0002 as a benchmarking aid, separate from 0001, so the thread does not ask core to keep these probes. The uprobe point is correct but argues against something the author did not propose for release.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9] is about dynamic prefix truncation during nbtree binary search, a different idea, and its regression reason does not carry over to split location choice. The thread does not mention any earlier attempt to randomize split location, so no prior rejection reason applies.
- standards-compat: 2 objection(s). No SQL syntax, GUC, catalog, pg_dump or wire change is proposed. Physical standbys replay the primary's split choices through WAL, so they keep the same layout.
- replication-ondisk: 0 objection(s). The proposal changes only which split point the primary picks among points nbtree already allows, so the on-disk page format, pg_upgrade and replication are unaffected. The split WAL record already carries the chosen split point and replay does not recompute it, so the random choice cannot make a standby diverge. The patch adds no catalog changes, no protocol changes and no extension API changes. The USDT probes add no persistent state. The nbtree VACUUM bookkeeping invariant does not apply, because nothing runs at VACUUM time. The batch-extension prototype in message #10 has standby and FSM questions, but it is a separate patch.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which equal-penalty leaf split point nbtree picks, inside a bounded interval, and adds USDT probes that only OS-level tracing tools with host access can attach to; no SQL surface, privilege, view, log output or user-controlled code path is added. Today's split choice is deterministic, so an unprivileged user cannot shape split behaviour any better under the proposal.
- maintenance-fit: 2 objection(s). The main design has the support of the nbtree committer in message #13 and is limited to SPLIT_DEFAULT leaf splits, so I raise no fit or ownership objection. The VACUUM bookkeeping invariant does not apply, because the patch does not change VACUUM.
- already-exists: 1 objection(s). The core change has no existing alternative. Message #1 says fillfactor only shifts the waves, and no hook or extension can change the nbtree split location, so the only point from this angle is the benchmark-only probe patch.
- demand: 2 objection(s). Two engaged reviewers, one of them the nbtsplitloc.c author, support the idea in principle, so these are 'not now, show demand' points and not a rejection.

### Precedents retrieved

- P1: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - judged not relevant; reason summarised from its thread
- P2: [Split StdRdOptions into HeapOptions and ToastOptions](https://commitfest.postgresql.org/patch/2370/) (cf, Returned with feedback, 2021-10-04) - judged not relevant; reason summarised from its thread
- P3: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P4: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P5: [function pg_setting_value_split() to parse shared_preload_libraries etc.](https://commitfest.postgresql.org/patch/2782/) (cf, Withdrawn, 2020-11-29) - judged not relevant; reason summarised from its thread
- P6: [array_random](https://commitfest.postgresql.org/patch/5881/) (cf, Withdrawn, 2026-01-06) - judged not relevant; reason summarised from its thread
- P7: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P8: [Review of arc4random for strong randomness](https://commitfest.postgresql.org/patch/1382/) (cf, Rejected, 2018-01-12) - judged not relevant; reason summarised from its thread
- P9: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - used in the review; reason summarised from its thread
- P10: [pgbench - add random_zipfian function](https://commitfest.postgresql.org/patch/1206/) (cf, Returned with feedback, 2018-06-26) - judged not relevant
- P11: [Revert "Replace PostmasterRandom() with a stronger way of generating randomness."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faae1c918e8aaae034eaf3ea103fcb6ba9adc5ab) (revert, Reverted, 2016-10-18) - judged not relevant
- P12: [Revert "Temporarily add some probes of tenk1's relallvisible in create_index.sql."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=91998539b227dfc6dd091714da7d106f2c95a321) (revert, Reverted, 2022-04-15) - judged not relevant
- P13: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P14: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant
- P15: [Use ReadRecentBuffer() for btree root page](https://commitfest.postgresql.org/patch/4419/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [Cube extension GiST penalty function improvement](https://commitfest.postgresql.org/patch/782/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P17: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P18: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant
- P19: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant
- P20: [Revert "nbtree: Remove useless row compare arg."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd2ce3792754e0705884aed15d1fd2ae43cc1500) (revert, Reverted, 2025-06-11) - judged not relevant
- P21: [Using indexUnchanged with nbtree](https://commitfest.postgresql.org/patch/3191/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P23: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P24: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 173 s; cost reported by the CLI: $2.33
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:06:59.386Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
