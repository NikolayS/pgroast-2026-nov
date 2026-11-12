# Design pre-review: [PATCH] Add parallel instrumentation hooks for CustomScan providers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6524](https://commitfest.postgresql.org/patch/6524/); [thread](https://www.postgresql.org/message-id/flat/CAGCUe0Kp7kTUWFjHhiE-xjZ4%3DUDe2fUdDVajKcjV%2BBT%2BT5d21A%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and a reviewer who knows the area accepts it as parity with built-in scans. Three fixable design points remain open from message #13. In v4 the provider picks its own shm_toc key (replication-ondisk-1), a provider cannot print per-worker output as built-in scans do (standards-compat-3), and no in-tree test runs the new path (maintenance-fit-2). The callback signatures should be settled before extensions build against them.

Not checked: The code was not read, so the exact call sites in execParallel.c, the behaviour on rescan and the conditions under which core calls each callback could not be checked. The points from messages #4 to #13 rest on the brief's summary of them.

## Prior attempts

- [asynchronous execution support for Custom Scan](https://www.postgresql.org/message-id/flat/CAJuF6cOMdvHSe+_7eHxzHGB3CVC9a890gBLLbpZTvi2pGgnB2g@mail.gmail.com) (related; Returned with feedback, 2024-02-01). "Since the author or no one else showed interest in taking it forward and the patch had no activity for more than 1 year, I have changed the status to RWF." ([message](https://www.postgresql.org/message-id/CALDaNm1Jek2D2ZELmL4dVjoHV%3DfUR750mPJYN1wZKmpyXiwp9w%40mail.gmail.com)) Different now: P2 added asynchronous execution callbacks to CustomScan, and this patch adds parallel instrumentation callbacks. Both extend CustomExecMethods with no in-tree consumer. P2 ended through inactivity, while this entry has an active reviewer who asks for test_extensible coverage in message #13.

## Objections a reviewer is likely to raise

### 1. Provider-chosen shm_toc keys can collide with core keys in the DSM

A provider calls shm_toc_insert with plan_node_id, or with a constant that another node or extension also uses. The leader stores two entries under one key, or a worker looks one up and gets the wrong chunk. The user sees a crash or wrong EXPLAIN ANALYZE numbers only under parallel plans.

In v4 core only checks each callback for NULL, so the provider does its own shm_toc_allocate and shm_toc_insert. plan_node_id is already taken by the parallel-scan coordinate, and the proposal gives no rule for picking another key. Message #13 raises this as the first open item, and the thread has not settled it. Core-owned allocation, as in ExecCustomScanInitializeDSM, keeps the key-space invariant in one place.

**Question:** Will core allocate and key the instrumentation chunk, so that a provider only gives a size and fills the memory?

<sub>fixable, level 2, ABI / extension API; messages [#13](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me); `replication-ondisk-1`</sub>

### 2. Per-worker EXPLAIN output cannot match built-in format

A provider retrieves per-worker metrics but cannot call ExplainOpenWorker or ExplainCloseWorker, so it writes its own worker grouping. Tools that parse EXPLAIN (FORMAT JSON) and expect the standard "Workers" array then miss or misread these metrics. Tool authors find out only when the parallel custom scan output differs from the output of other scans.

The commit message promises that providers can "fully support parallel EXPLAIN ANALYZE metrics" in a way similar to built-in scan nodes. Built-in nodes print worker details through static helpers in explain.c that extensions cannot reach (message #13, item 5). Without an exported helper or a new callback, each provider will invent its own output shape for the same kind of data. The proposal does not yet say which of these two options it takes.

**Question:** How will a provider print per-worker statistics in the same text and JSON structure that built-in parallel scans use?

<sub>fixable, level 2, design; messages [#13](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me); `standards-compat-3`</sub>

### 3. No in-tree user, so later executor changes can break this unnoticed

A later commit reworks parallel instrumentation in execParallel.c, as the recent SeqScan and IndexScan commits did. No core test calls the CustomScan callbacks, so check-world passes. An extension author finds the broken path after the release ships.

Message #1 says tests belong in the extension that uses the callbacks. That leaves core with a path that no buildfarm member exercises. Message #13 points to src/test/modules/test_extensible as the natural place for this test. Without it, whoever commits the patch owns a path that fails silently whenever execParallel.c changes.

**Question:** Can test_extensible implement all four callbacks and show worker instrumentation in EXPLAIN ANALYZE output?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAGCUe0Kp7kTUWFjHhiE-xjZ4%3DUDe2fUdDVajKcjV%2BBT%2BT5d21A%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me); `maintenance-fit-2`</sub>

## Questions for the author

1. Will core allocate the instrumentation chunk and own its shm_toc key, so that a provider only gives a size and fills the memory?
2. How will a provider print per-worker statistics in the same text and JSON structure as built-in parallel scans, given that ExplainOpenWorker and ExplainCloseWorker are static?
3. Can test_extensible implement all four callbacks and show worker instrumentation in EXPLAIN ANALYZE output?
4. Does core skip the four callbacks when the query does not request instrumentation?
5. Does RetrieveInstrumentationCustomScan run once at query end or on every rescan, and what must it copy out of shared memory?

## Touch points

- **ABI / extension API:** The patch adds four optional callbacks to CustomExecMethods in extensible.h.
- **backpatchability:** The CommitFest entry lists target "stable", but new CustomExecMethods members fit master only.
- **maintenance burden ("who maintains this in 10 years"):** execParallel.c gains a CustomScan instrumentation path that no core test runs in v4.

## Who would commit it, and when

- Owner: qualified committer engaged. Tomas Vondra reviewed v4 in message #13. The owner needs to know execParallel.c and the parallel instrumentation code for built-in scans.
- Priority: fix design first; audience niche; review cost low. Only authors of parallel custom scan providers gain, and the v4 patch is about 9 KiB in the executor and docs. The burden stays medium until an in-tree test covers the path, and message #13 lists design items a v5 must answer first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
