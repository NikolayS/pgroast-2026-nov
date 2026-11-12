# Design pre-review: Showing "Table Block Fetches" in EXPLAIN ANALYZE for index-only scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7467](https://commitfest.postgresql.org/patch/7467/) (Needs review); authors: Peter Geoghegan (pgeoghegan)
- Thread: [Showing "Table Block Fetches" in EXPLAIN ANALYZE for index-only scans](https://www.postgresql.org/message-id/flat/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), started 2026-10-15
- Messages read: 3 of 3
- Patch prose read: v1-0001-Show-block-not-tuple-fetches-in-EXPLAIN-ANALYZE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection disputes counting table blocks, and both replies support it. The two surviving objections, demand-1 and standards-compat-1, are fixable and concern the same point: version 1 removes the "Heap Fetches" line, which both reviewers want kept and which tools parse by key name. The author offered in message #1 to report both counts, so a next version can settle this.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how the block counter behaves across rescans and parallel workers. It also could not check what the changes in nodeIndexscan.c and nodeBitmapIndexscan.c do.

## The proposal, restated

**Problem.** EXPLAIN ANALYZE shows "Heap Fetches" for index-only scans, which counts tuples whose visibility had to be checked in the table. The author says this is only a rough proxy for the real cost to users of heap pages not being all-visible in the visibility map.

**Proposed design.** The patch replaces "Heap Fetches" with "Table Block Fetches", the number of times the scan switched to a different table block, which is one ReadBuffer call per fetch. The counter lives in heapam, made easier by commit ddce1da5. Version 1 drops the tuple count, but the author offers to report both.

**User-visible surface.**
- EXPLAIN ANALYZE output for Index Only Scan nodes: "Heap Fetches" line removed, "Table Block Fetches" line added.
- IndexScanInstrumentation-related struct in instrument_node.h changes, visible to extensions that read executor instrumentation.

**What the author claims.**
- Block fetches show the exact physical cost of pages not being all-visible, not a rough proxy.
- The count is table AM agnostic, at least in theory.
- Repeated rescans that touch only the same heap page report a single block fetch.
- The current tuple-based counter may exist mainly because a block count was awkward to do in nodeIndexonlyscan.c before ddce1da5.

**Thread state.** Sami Imseih and Greg Burd support counting blocks but both want a tuple count kept, with Sami suggesting "Table Tuple Fetches" and exposure in pg_stat_*_indexes. Greg also asks whether the same fields could later appear on plain Index Scan nodes; the author has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No one in the thread asks to remove the Heap Fetches line

fixable · level 1b, rubric item 4 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A DBA watches "Heap Fetches" in auto_explain logs or EXPLAIN JSON to tell when VACUUM is falling behind on an index-only scan. After upgrading, the key is gone and their check finds nothing. The scan that used to show many fetched tuples on a few pages now shows a small block count, so the alert stays quiet.

Both replies want a block count, and both also want the tuple count kept (messages #2 and #3). Message #3 gives a case where only the two counters together show stale entries. Removing the line costs every existing reader of EXPLAIN output, while the gain is a shorter plan. The author offers to report both in message #1, but v1 still removes the line.

Thread: [#1 (Peter Geoghegan, 2026-10-15)](https://www.postgresql.org/message-id/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), [#2 (Sami Imseih, 2026-10-15)](https://www.postgresql.org/message-id/CAN12%2BYKuDHUSJEApCK-Wk3fjj8aRWWCEs8BYPxxzSuHgpw9t8g%40mail.gmail.com), [#3 (Greg Burd, 2026-10-29)](https://www.postgresql.org/message-id/auOuiXb0_0kI1rCL%40floki)

**Question:** Which users or workloads gain from removing the tuple count, as opposed to adding a block count next to it?

Counter-check: held. Both replies ask to keep the tuple count, and message #3 gives a case that needs both counters. The author's offer depends on consensus and version 1 still drops the count, so the material does not yet answer this.

### 2. Removing the "Heap Fetches" key breaks existing EXPLAIN consumers

fixable · level 2, rubric item 12 · design, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A monitoring tool or auto_explain log parser reads "Heap Fetches" from FORMAT JSON output to flag tables whose visibility map needs vacuum. After upgrade the key is absent, so the check finds nothing and reports every index-only scan as clean. The DBA finds out only when someone notices the alert stopped firing.

"Heap Fetches" has long been the standard signal that index-only scans are hitting the heap, and the JSON, XML and YAML formats exist so that tools can depend on key names. Version 1 drops the key, and Sami's suggested "Table Tuple Fetches" in message #2 would rename it, which still breaks those parsers. Messages #2 and #3 ask to keep a tuple count, but neither says it should keep its current name. A committer would likely prefer adding the new line and keeping "Heap Fetches" unchanged unless a rename is argued separately.

Thread: [#1 (Peter Geoghegan, 2026-10-15)](https://www.postgresql.org/message-id/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), [#2 (Sami Imseih, 2026-10-15)](https://www.postgresql.org/message-id/CAN12%2BYKuDHUSJEApCK-Wk3fjj8aRWWCEs8BYPxxzSuHgpw9t8g%40mail.gmail.com), [#3 (Greg Burd, 2026-10-29)](https://www.postgresql.org/message-id/auOuiXb0_0kI1rCL%40floki)

**Question:** Why should the tuple counter not stay under its existing "Heap Fetches" key, with "Table Block Fetches" added beside it?

Counter-check: held. Version 1 removes the "Heap Fetches" key, and the proposal does not say how tools that parse the structured formats should cope. The author's offer to report both counts does not settle whether the existing key name stays.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes the index scan instrumentation struct in instrument_node.h, which table AMs fill and extensions can read. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Removing the "Heap Fetches" key from text, JSON, XML and YAML output puts a change on every tool that parses it. |
| performance on the hot path | **yes** | The patch changes counter maintenance in heapam_indexscan.c, which runs during index-only scans; no cost concern survived review. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No one in the thread asks to remove the Heap Fetches line |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version keep the tuple count under the existing "Heap Fetches" key and add "Table Block Fetches" beside it, and if not, why?
2. Which users or workloads gain from removing the tuple count, as opposed to adding the block count next to it?
3. How does a parallel index-only scan add up table block fetches across workers, given that two workers can fetch the same block?
4. Why does the patch touch nodeIndexscan.c and nodeBitmapIndexscan.c, and does EXPLAIN output for those nodes change?
5. Are the same instrumentation fields meant to be shown later on plain Index Scan nodes, filled by the table AM, as message #3 asks?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The author states in message #1 that he committed ddce1da5, the table AM index scan interface this patch builds on.
- **Liability:** The owner signs for a permanent change to EXPLAIN ANALYZE key names that external tools parse, and for the meaning of an instrumentation counter that each table AM must maintain.
- **Author follow-up:** The author offers in message #1 to produce a revision that reports both counts; he has not yet replied to messages #2 and #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 25 KiB, mostly EXPLAIN code, docs and expected test output, and two reviewers support the block count. The open point is whether "Heap Fetches" stays, which a second version can settle.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` New "Table Block" wording departs from existing "Heap Blocks" output. Refuted (no failure scenario): "Heap Blocks" on a Bitmap Heap Scan counts exact and lossy bitmap pages, a different quantity, so the scenario of a failed comparison is not a failure. Message #1 gives table AM neutrality as the reason and leaves the name open.
- `maintenance-fit-1` Removing the "Heap Fetches" key breaks tools that parse EXPLAIN output. Refuted (not specific): Duplicate of standards-compat-1, which states the same parser break and asks the sharper question about keeping the existing key name.
- `maintenance-fit-2` Table AMs that do not maintain the new counter silently report zero. Refuted (not specific): Message #1 says commit ddce1da5 already made the table AM responsible for the "Heap Fetches" counter. An AM that does not count already shows zero today, so the patch does not create this gap.

### Review angles

- prior-attempts: 0 objection(s). No precedent found for this idea. [P19] adds an EXPLAIN counter for bitmap heap scans, a different node and a different counter, and it records no reason, so nothing from it applies here. The thread does not point to any earlier discussion of block-level fetch counts for index-only scans.
- standards-compat: 2 objection(s). The idea of counting table blocks is not disputed from this angle. The objections concern only the output contract and naming.
- replication-ondisk: 0 objection(s). The patch changes only EXPLAIN ANALYZE output and an in-memory instrumentation struct. It writes nothing to disk, WAL, catalogs or the wire protocol, so there is nothing for pg_upgrade, standbys or subscribers to handle. The instrument_node.h change goes into master only, and extensions already rebuild for each major version. Parallel workers share that struct within one server binary. Exposing the counts in pg_stat_*_indexes, as message #2 suggests, would add pgstat state, but that is not part of this patch.
- security: 0 objection(s). The proposal changes which counter EXPLAIN ANALYZE shows for index-only scans. The block count reaches the same user, through the same command, who can already run the query and see its buffer counts, so it adds no new privilege, exposure or denial-of-service path. The suggested pg_stat_*_indexes exposure is not part of v1.
- maintenance-fit: 2 objection(s). Two fixable objections only. The per-block counter adds no meaningful hot-path cost, and a committer who knows the area wrote the patch.
- already-exists: 0 objection(s). No existing feature gives per-node table block fetches for an index-only scan. EXPLAIN (BUFFERS) mixes index, visibility map and heap buffers in one number, and pg_statio_user_tables heap_blks_hit/read are per table and cumulative, so neither can isolate this scan node. No extension or external tool that does this is known with certainty, and the counter needs table AM internals, so it cannot easily live outside core. [P19] is about bitmap heap scans, has no recorded reason, and does not apply.
- demand: 1 objection(s). The block count itself has support from two independent reviewers and a concrete cost case in message #2. The only demand gap is the removal of the existing tuple count.

### Precedents retrieved

- P1: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P2: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P3: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P4: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P5: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P6: [explain analyze rows=%.0f](https://commitfest.postgresql.org/patch/3704/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P7: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P8: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P9: [Sampling-based timing for EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/4100/) (cf, Returned with feedback, 2023-07-03) - judged not relevant; reason summarised from its thread
- P10: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P11: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P14: [Revert FETCH/MOVE int64 patch. Was using incorrect checks for fetch/move in scan.l.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0e20c485617c3bcde8d9f0759d5ee1ef1c442004) (revert, Reverted, 2006-09-03) - judged not relevant
- P15: [Change EXPLAIN's ANALYZE option to the clearer (EXEC)](https://commitfest.postgresql.org/patch/2121/) (cf, Withdrawn, 2019-07-02) - judged not relevant
- P16: [Output affected rows in EXPLAIN](https://commitfest.postgresql.org/patch/4551/) (cf, Rejected, 2023-11-22) - judged not relevant
- P17: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant
- P18: [Allow EXPLAIN to Output Page Fault Information](https://commitfest.postgresql.org/patch/5477/) (cf, Withdrawn, 2025-02-08) - judged not relevant
- P19: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - used in the review
- P20: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P21: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P22: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P23: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P24: [Parallel leader process info in EXPLAIN output](https://commitfest.postgresql.org/patch/2315/) (cf, Returned with feedback, 2020-03-17) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 90 s; cost reported by the CLI: $0.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:58:31.381Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
