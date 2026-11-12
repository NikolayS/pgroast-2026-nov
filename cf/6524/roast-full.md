# Design pre-review: [PATCH] Add parallel instrumentation hooks for CustomScan providers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6524](https://commitfest.postgresql.org/patch/6524/) (Needs review); authors: Siddharth Kothari (sidkot.google.com)
- Thread: [\[PATCH\] Add RetrieveInstrumentation hook for CustomScan providers](https://www.postgresql.org/message-id/flat/CAGCUe0Kp7kTUWFjHhiE-xjZ4%3DUDe2fUdDVajKcjV%2BBT%2BT5d21A%40mail.gmail.com), started 2026-02-18
- Messages read: 13 of 13
- Patch prose read: v4-0001-Add-parallel-instrumentation-hooks-for-CustomScan.patch, 0001-Add-RetrieveInstrumentationCustomScan-hook-for-Custo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and a reviewer who knows the area accepts it as parity with built-in scans. Three fixable design points remain open from message #13. In v4 the provider picks its own shm_toc key (replication-ondisk-1), a provider cannot print per-worker output as built-in scans do (standards-compat-3), and no in-tree test runs the new path (maintenance-fit-2). The callback signatures should be settled before extensions build against them.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact call sites in execParallel.c, the behaviour on rescan and the conditions under which core calls each callback could not be checked. The points from messages #4 to #13 rest on the brief's summary of them.

## The proposal, restated

**Problem.** CustomScan providers have no standard way to copy instrumentation from parallel workers to the leader before the DSM segment goes away. So extension authors cannot report per-worker or aggregated metrics in EXPLAIN ANALYZE for parallel custom scans. Built-in scans such as SeqScan and IndexScan got separate instrumentation DSM handling, and CustomScan did not.

**Proposed design.** The patch adds four optional callbacks to CustomExecMethods for estimating, setting up and retrieving instrumentation shared memory. execParallel.c calls them for CustomScanState nodes even when the node is not parallel_aware, like built-in scans do for EXPLAIN ANALYZE. In v4 the provider does the shared memory allocation itself inside the callbacks, and core only checks each callback for NULL.

**User-visible surface.**
- C API: new optional callback EstimateDSMCustomScanInstrumentation in CustomExecMethods.
- C API: new optional callback InitializeDSMCustomScanInstrumentation in CustomExecMethods.
- C API: new optional callback InitializeWorkerCustomScanInstrumentation in CustomExecMethods.
- C API: new optional callback RetrieveInstrumentationCustomScan in CustomExecMethods.
- Docs: custom-scan.sgml describes the four callbacks.

**What the author claims.**
- Existing extensions see no change and no overhead, because each callback is checked for NULL.
- The patch follows the same pattern as recent core commits for SeqScan and IndexScan instrumentation.
- Tests belong in an extension that implements the callbacks, not in core regression tests.
- The patch passes make check-world and is not platform-specific.

**Thread state.** Tomas Vondra and Dilip Kumar find the idea fine, and Dilip approved v3. Tomas's review of v4 lists open items: core should own the shm_toc allocation and keys, callbacks run without instrumentation requested, test_extensible tests, docs detail, per-worker EXPLAIN access, and callback naming.

## 2. Prior attempts

### asynchronous execution support for Custom Scan (related)

- **Link:** [https://commitfest.postgresql.org/patch/3813/](https://commitfest.postgresql.org/patch/3813/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJuF6cOMdvHSe+_7eHxzHGB3CVC9a890gBLLbpZTvi2pGgnB2g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJuF6cOMdvHSe+_7eHxzHGB3CVC9a890gBLLbpZTvi2pGgnB2g@mail.gmail.com)
- **Date:** decided 2024-02-01; started 2022-08-13
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The entry died of inactivity: no one reviewed the patch in depth, and the author did not move it forward. Etsuro Fujita said he would review it once an example extension existed, but the visible messages show no review after that. Vignesh C returned it with feedback after more than a year with no activity.
- **Reason (quote):** "Since the author or no one else showed interest in taking it forward and the patch had no activity for more than 1 year, I have changed the status to RWF." ([message](https://www.postgresql.org/message-id/CALDaNm1Jek2D2ZELmL4dVjoHV%3DfUR750mPJYN1wZKmpyXiwp9w%40mail.gmail.com))
- **What is different this time:** P2 added asynchronous execution callbacks to CustomScan, and this patch adds parallel instrumentation callbacks. Both extend CustomExecMethods with no in-tree consumer. P2 ended through inactivity, while this entry has an active reviewer who asks for test_extensible coverage in message #13.


## 3. Objections a reviewer is likely to raise

### 1. Provider-chosen shm_toc keys can collide with core keys in the DSM

fixable · level 2, rubric item 7 · ABI / extension API · `replication-ondisk-1`

A provider calls shm_toc_insert with plan_node_id, or with a constant that another node or extension also uses. The leader stores two entries under one key, or a worker looks one up and gets the wrong chunk. The user sees a crash or wrong EXPLAIN ANALYZE numbers only under parallel plans.

In v4 core only checks each callback for NULL, so the provider does its own shm_toc_allocate and shm_toc_insert. plan_node_id is already taken by the parallel-scan coordinate, and the proposal gives no rule for picking another key. Message #13 raises this as the first open item, and the thread has not settled it. Core-owned allocation, as in ExecCustomScanInitializeDSM, keeps the key-space invariant in one place.

Thread: [#13 (Tomas Vondra, 2026-11-11)](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me)

**Question:** Will core allocate and key the instrumentation chunk, so that a provider only gives a size and fills the memory?

Counter-check: held. In v4 the provider allocates and picks the shm_toc key, and plan_node_id is already taken by the parallel scan chunk. Message #13 item 1 raises this and v4 has no answer, but a new version can move allocation into core.

### 2. Per-worker EXPLAIN output cannot match built-in format

fixable · level 2, rubric item 12 · design, ABI / extension API · `standards-compat-3`

A provider retrieves per-worker metrics but cannot call ExplainOpenWorker or ExplainCloseWorker, so it writes its own worker grouping. Tools that parse EXPLAIN (FORMAT JSON) and expect the standard "Workers" array then miss or misread these metrics. Tool authors find out only when the parallel custom scan output differs from the output of other scans.

The commit message promises that providers can "fully support parallel EXPLAIN ANALYZE metrics" in a way similar to built-in scan nodes. Built-in nodes print worker details through static helpers in explain.c that extensions cannot reach (message #13, item 5). Without an exported helper or a new callback, each provider will invent its own output shape for the same kind of data. The proposal does not yet say which of these two options it takes.

Thread: [#13 (Tomas Vondra, 2026-11-11)](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me)

**Question:** How will a provider print per-worker statistics in the same text and JSON structure that built-in parallel scans use?

Counter-check: held. ExplainOpenWorker and ExplainCloseWorker are static in explain.c, and the commit message promises parity with built-in scans. Message #13 item 5 raises this and no reply answers it yet.

### 3. No in-tree user, so later executor changes can break this unnoticed

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A later commit reworks parallel instrumentation in execParallel.c, as the recent SeqScan and IndexScan commits did. No core test calls the CustomScan callbacks, so check-world passes. An extension author finds the broken path after the release ships.

Message #1 says tests belong in the extension that uses the callbacks. That leaves core with a path that no buildfarm member exercises. Message #13 points to src/test/modules/test_extensible as the natural place for this test. Without it, whoever commits the patch owns a path that fails silently whenever execParallel.c changes.

Thread: [#1 (Siddharth Kothari, 2026-02-18)](https://www.postgresql.org/message-id/CAGCUe0Kp7kTUWFjHhiE-xjZ4%3DUDe2fUdDVajKcjV%2BBT%2BT5d21A%40mail.gmail.com), [#13 (Tomas Vondra, 2026-11-11)](https://www.postgresql.org/message-id/3978c569-954f-44c1-bf53-db7a680cf0d4%40vondra.me)

**Question:** Can test_extensible implement all four callbacks and show worker instrumentation in EXPLAIN ANALYZE output?

Counter-check: held. Message #1 puts tests outside core, so no buildfarm member would run the new execParallel.c path. Message #13 item 4 asks for test_extensible coverage and the thread has no reply yet.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds four optional callbacks to CustomExecMethods in extensible.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry lists target "stable", but new CustomExecMethods members fit master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | execParallel.c gains a CustomScan instrumentation path that no core test runs in v4. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will core allocate the instrumentation chunk and own its shm_toc key, so that a provider only gives a size and fills the memory?
2. How will a provider print per-worker statistics in the same text and JSON structure as built-in parallel scans, given that ExplainOpenWorker and ExplainCloseWorker are static?
3. Can test_extensible implement all four callbacks and show worker instrumentation in EXPLAIN ANALYZE output?
4. Does core skip the four callbacks when the query does not request instrumentation?
5. Does RetrieveInstrumentationCustomScan run once at query end or on every rescan, and what must it copy out of shared memory?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tomas Vondra reviewed v4 in message #13. The owner needs to know execParallel.c and the parallel instrumentation code for built-in scans.
- **Liability:** The owner signs for the four callback signatures as an extension API for the life of the release, and for keeping the CustomScan path working when execParallel.c changes.
- **Author follow-up:** The author posted new versions up to v4 after review and kept the thread alive for nine months. No reply to message #13 is visible yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** Only authors of parallel custom scan providers gain, and the v4 patch is about 9 KiB in the executor and docs. The burden stays medium until an in-tree test covers the path, and message #13 lists design items a v5 must answer first.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` CommitFest target "stable" implies an ABI break in CustomExecMethods. Refuted (no failure scenario): Nobody in the thread proposes a backpatch, and the entry moved from PG19-Final into the PG20 CommitFests, so the patch targets master. Postgres does not backpatch new extension APIs, so the minor-release crash cannot happen.
- `standards-compat-2` Instrumentation callbacks break the existing parallel callback contract. Refuted (not specific): Duplicate of replication-ondisk-1, which states the shm_toc key problem more directly. The naming point is a small item that message #13 already lists.
- `replication-ondisk-2` New CustomExecMethods fields break ABI if backpatched to a stable branch. Refuted (no failure scenario): Same point as standards-compat-1, and it fails for the same reason: the thread shows no backpatch plan. The objection itself says the change is normal major-version churn on master.
- `maintenance-fit-1` Providers choose shm_toc keys and allocate the instrumentation DSM themselves. Refuted (not specific): Duplicate of replication-ondisk-1.
- `maintenance-fit-3` API may be incomplete: per-worker EXPLAIN output and ForeignScan parity. Refuted (not specific): Duplicate of standards-compat-3 for the per-worker EXPLAIN gap. Message #13 calls ForeignScan "a separate patch", so that part is not a defect of this proposal.
- `demand-1` No named custom scan provider shows a need for four new callbacks. Refuted (already handled): The leader's ShutdownCustomScan runs before Gather waits for its workers, so it cannot collect their final counters. A committer who knows the area accepts the goal as parity with built-in scans in message #5, and the cost falls only on providers that set the callbacks.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed this idea. P8 covers extension EXPLAIN options and its reason, that a hook already exists, does not fit: no existing callback reaches worker data for a CustomScan that is not parallel_aware. P1, P15 and P17 record no reason that applies, and the open design points are already listed in message #13.
- standards-compat: 3 objection(s). The idea itself does not conflict with SQL, pg_dump, psql or GUC conventions. The objections concern the C API contract only.
- replication-ondisk: 2 objection(s). The patch adds no on-disk, WAL, catalog or wire-protocol state. The only cross-process state is the per-query DSM, which is torn down at query end.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds optional C callbacks that only a loaded C extension can set, and loading one already needs superuser. It adds no SQL surface, privilege or user-visible data path, so no unprivileged user gains new access. The shm_toc key clash in message #13 is a crash risk for the provider's own code. That belongs to the design angle.
- maintenance-fit: 3 objection(s). The idea fits core: it closes a gap left by recent SeqScan and IndexScan commits, and a committer who knows the area is engaged. The objections concern the API shape before release.
- already-exists: 0 objection(s). ShutdownCustomScan already runs before the parallel DSM segment goes away, and the docs name it for this use. It works only for parallel-aware nodes, though, because InitializeDSMCustomScan is skipped otherwise. Message #5 shows the non-parallel-aware case cannot be handled today, and a committer asked for the extra callbacks. The proposal still does not say why ShutdownCustomScan is not enough for parallel-aware nodes.
- demand: 1 objection(s). Demand is thin but not absent. A committer who knows execParallel.c treats this as parity with the built-in scans, and only providers that implement the callbacks pay the cost, so the objection above asks for a real consumer and is not a call to reject.

### Precedents retrieved

- P1: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - used in the review
- P2: [asynchronous execution support for Custom Scan](https://commitfest.postgresql.org/patch/3813/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P3: [ctidscan as an example of custom-scan](https://commitfest.postgresql.org/patch/18/) (cf, Rejected, 2015-07-22) - judged not relevant; reason summarised from its thread
- P4: [Free memory allocated by waitonlock_error_callback()](https://commitfest.postgresql.org/patch/6069/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P5: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant; reason summarised from its thread
- P6: [Revert "Refactor CopyReadAttributes{CSV,Text}() to use a callback in COPY FROM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=06bd311bce24083c76d9741ae89c98750aaf4b41) (revert, Reverted, 2024-02-14) - judged not relevant
- P7: [Revert "Refactor ChangeVarNodesExtended() using the custom callback"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2782f3b8454676c9df408aa0bc960efca0f3eb22) (revert, Reverted, 2025-05-03) - judged not relevant
- P8: [Make it possible to add custom options to EXPLAIN](https://commitfest.postgresql.org/patch/4643/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P9: [Revert "Temporarily instrument postgres_fdw test to look for statistics changes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=04e7ecadf64d18f67ca1d0632d8ab71f120ca5e3) (revert, Reverted, 2018-03-08) - judged not relevant
- P10: [Support custom authentication methods using hooks](https://commitfest.postgresql.org/patch/3546/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P11: [Revert "Add some temporary instrumentation to pgstat.c."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5a2bb060121f574d6587e5fc6c7f1d4e3a759351) (revert, Reverted, 2012-05-15) - judged not relevant
- P12: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P13: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P14: [Xact event callback at command start](https://commitfest.postgresql.org/patch/2860/) (cf, Rejected, 2021-09-01) - judged not relevant; reason summarised from its thread
- P15: [Parallel-aware file_fdw](https://commitfest.postgresql.org/patch/2108/) (cf, Returned with feedback, 2019-09-30) - used in the review
- P16: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P17: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - used in the review
- P18: [Revert "Only provide new libpq sslpasskey hook for openssl-enabled builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6741cfa5964768279a13e01e46931664fe0903a4) (revert, Reverted, 2020-04-17) - judged not relevant
- P19: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [explain analyze rows=%.0f](https://commitfest.postgresql.org/patch/3704/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P21: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant
- P22: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P23: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P24: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 172 s; cost reported by the CLI: $2.69
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 94 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:53:44.516Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
