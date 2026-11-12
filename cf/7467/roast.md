# Design pre-review: Showing "Table Block Fetches" in EXPLAIN ANALYZE for index-only scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7467](https://commitfest.postgresql.org/patch/7467/); [thread](https://www.postgresql.org/message-id/flat/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection disputes counting table blocks, and both replies support it. The two surviving objections, demand-1 and standards-compat-1, are fixable and concern the same point: version 1 removes the "Heap Fetches" line, which both reviewers want kept and which tools parse by key name. The author offered in message #1 to report both counts, so a next version can settle this.

Not checked: Without the code, this review could not check how the block counter behaves across rescans and parallel workers. It also could not check what the changes in nodeIndexscan.c and nodeBitmapIndexscan.c do.

## Prior attempts

None found before 2026-10-15.

## Objections a reviewer is likely to raise

### 1. No one in the thread asks to remove the Heap Fetches line

A DBA watches "Heap Fetches" in auto_explain logs or EXPLAIN JSON to tell when VACUUM is falling behind on an index-only scan. After upgrading, the key is gone and their check finds nothing. The scan that used to show many fetched tuples on a few pages now shows a small block count, so the alert stays quiet.

Both replies want a block count, and both also want the tuple count kept (messages #2 and #3). Message #3 gives a case where only the two counters together show stale entries. Removing the line costs every existing reader of EXPLAIN output, while the gain is a shorter plan. The author offers to report both in message #1, but v1 still removes the line.

**Question:** Which users or workloads gain from removing the tuple count, as opposed to adding a block count next to it?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN12%2BYKuDHUSJEApCK-Wk3fjj8aRWWCEs8BYPxxzSuHgpw9t8g%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/auOuiXb0_0kI1rCL%40floki); `demand-1`</sub>

### 2. Removing the "Heap Fetches" key breaks existing EXPLAIN consumers

A monitoring tool or auto_explain log parser reads "Heap Fetches" from FORMAT JSON output to flag tables whose visibility map needs vacuum. After upgrade the key is absent, so the check finds nothing and reports every index-only scan as clean. The DBA finds out only when someone notices the alert stopped firing.

"Heap Fetches" has long been the standard signal that index-only scans are hitting the heap, and the JSON, XML and YAML formats exist so that tools can depend on key names. Version 1 drops the key, and Sami's suggested "Table Tuple Fetches" in message #2 would rename it, which still breaks those parsers. Messages #2 and #3 ask to keep a tuple count, but neither says it should keep its current name. A committer would likely prefer adding the new line and keeping "Heap Fetches" unchanged unless a rename is argued separately.

**Question:** Why should the tuple counter not stay under its existing "Heap Fetches" key, with "Table Block Fetches" added beside it?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAH2-Wz%3DvLkCGiLs-9kB0ubZAUyYnrMu_GN78TJ%3Du2sVncJeanA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN12%2BYKuDHUSJEApCK-Wk3fjj8aRWWCEs8BYPxxzSuHgpw9t8g%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/auOuiXb0_0kI1rCL%40floki); `standards-compat-1`</sub>

## Questions for the author

1. Will the next version keep the tuple count under the existing "Heap Fetches" key and add "Table Block Fetches" beside it, and if not, why?
2. Which users or workloads gain from removing the tuple count, as opposed to adding the block count next to it?
3. How does a parallel index-only scan add up table block fetches across workers, given that two workers can fetch the same block?
4. Why does the patch touch nodeIndexscan.c and nodeBitmapIndexscan.c, and does EXPLAIN output for those nodes change?
5. Are the same instrumentation fields meant to be shown later on plain Index Scan nodes, filled by the table AM, as message #3 asks?

## Touch points

- **ABI / extension API:** The patch changes the index scan instrumentation struct in instrument_node.h, which table AMs fill and extensions can read.
- **maintenance burden ("who maintains this in 10 years"):** Removing the "Heap Fetches" key from text, JSON, XML and YAML output puts a change on every tool that parses it.
- **performance on the hot path:** The patch changes counter maintenance in heapam_indexscan.c, which runs during index-only scans; no cost concern survived review.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No one in the thread asks to remove the Heap Fetches line

## Who would commit it, and when

- Owner: qualified committer engaged. The author states in message #1 that he committed ddce1da5, the table AM index scan interface this patch builds on.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 25 KiB, mostly EXPLAIN code, docs and expected test output, and two reviewers support the block count. The open point is whether "Heap Fetches" stays, which a second version can settle.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
