# Design pre-review: Add storage I/O tracking to EXPLAIN IO option

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5526](https://commitfest.postgresql.org/patch/5526/) (Needs review); authors: Atsushi Torikoshi (atsushitorikoshi)
- Thread: [RFC: Allow EXPLAIN to Output Page Fault Information](https://www.postgresql.org/message-id/flat/c20f6340eb26f3b736abc59471bfada8%40oss.nttdata.com), started 2024-12-24
- Messages read: 50 of 50
- Patch prose read: v13-0001-Add-storage-I-O-tracking-to-IO-option.patch, v1-0001-PoC-Allow-EXPLAIN-to-output-page-fault-info.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, but v13 shows no Storage I/O output under io_method = worker, the default (maintenance-fit-1). Andres said in message #26 that the approach then fails where most users run, and nothing after v12 shows that he or Tom accepts hiding the output (prior-attempts-1). The thread also does not weigh an extension plus a small parallel-worker hook (already-exists-2). The level 2 objections were not evaluated.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check the getrusage() overhead in parallel workers or whether the io_method gating covers every path. The level 2 objections on platform units, zero versus not measured, and future I/O changes were not evaluated.

## The proposal, restated

**Problem.** EXPLAIN BUFFERS shows shared buffer hits and reads, but not whether a read was served by the OS page cache or by storage. People who diagnose slow queries, especially where shared_buffers is much smaller than RAM, cannot tell if a query is really disk-bound.

**Proposed design.** The backend calls getrusage() at the start and end of planning and of execution and reports the difference in ru_inblock and ru_oublock. Parallel workers send their counts to the leader, as is done for buffer and WAL usage. The IO option enables the output, so ANALYZE is required, and nothing is shown when io_method = worker.

**User-visible surface.**
- New "Storage I/O: read=N write=N" lines in the Planning and Execution sections of EXPLAIN (ANALYZE, IO) text output.
- New "Storage I/O Read" and "Storage I/O Write" properties in JSON, XML and YAML EXPLAIN output.
- No Storage I/O output when io_method = worker; values are always zero on Windows.
- Changes to the Windows getrusage() port header (src/include/port/win32/sys/resource.h).
- Documented auto_explain support is planned as a follow-up and is not in this patch.

**What the author claims.**
- Storage I/O is much slower than shared buffer or OS cache hits, so these counts show whether storage I/O is the bottleneck.
- Calling getrusage() only at phase boundaries has small overhead; per-node calls were tried and were too slow.
- The counters work with io_method = sync and io_uring; Andres's test showed correct counts with io_uring.
- pg_stat_kcache has used the same counters for years without problem reports (Julien Rouhaud).
- io_uring adoption is growing; RHEL 10.2 moved it out of Technology Preview.

**Thread state.** Andres Freund and Tom Lane objected that the counts miss I/O done by worker-based AIO, bgwriter and checkpointer; Lukas Fittl asked whether there is a path forward when most installations use I/O workers. Jelte supports the patch; v13 is a rebase with no reply after it.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-12-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Counters vanish under the default io_method, where most users run

fixable · level 1b, rubric item 4 · desirability, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-1`

A user on a managed service, where only I/O workers are offered, runs EXPLAIN (ANALYZE, IO) to check whether a slow query is disk-bound. No Storage I/O line appears. The core feature gives that user nothing, but the project still carries its parallel plumbing, platform quirks and tests.

Message #39 says worker is now the default io_method. Message #46 reports that managed providers offer only I/O workers and that containers often lack io_uring. The author answers that sync and io_uring users still benefit and that io_uring adoption is growing (message #48, #49). A committer would likely ask whether a core EXPLAIN feature should rest on that forecast, since the cost lands on every release while the benefit reaches only a minority of installations.

Thread: [#26 (Andres Freund, 2025-02-11)](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#39 (torikoshia, 2026-01-28)](https://www.postgresql.org/message-id/a8fb87e49a5a547519bb916e375ab563%40oss.nttdata.com), [#46 (Lukas Fittl, 2026-05-25)](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com), [#48 (Atsushi Torikoshi, 2026-06-01)](https://www.postgresql.org/message-id/CAM6-o%3DCATBQpDQAa3akuKqw9gWfpN-N7nVr9%3D8qSejKV%2BQyHXQ%40mail.gmail.com), [#49 (torikoshia, 2026-08-19)](https://www.postgresql.org/message-id/95ff66682bec0d10e18ea9c6a3dd3f93%40oss.nttdata.com)

**Question:** What share of Postgres 19+ installations can use sync or io_uring today, and what evidence shows those users ask for this in EXPLAIN?

Counter-check: held. Message #38 confirms worker is the default and message #46 reports providers that offer only workers. The author's answer in messages #48 and #49 is a forecast of io_uring adoption without numbers, so the gain reaches an unmeasured minority.

### 2. Earlier in-thread objection about I/O workers still stands for v13

fixable · level 1b, rubric item 3 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

An operator on a managed service that offers only io_method = worker runs EXPLAIN (ANALYZE, IO) on a slow query and gets no Storage I/O line. Others compare numbers taken under sync with numbers taken under worker and misread the difference. The bug reports go to the AIO maintainers.

In message #17 and message #26 Andres objected that the approach cannot work with worker-based AIO, the expected default, and that he would be on the hook for the bug reports. Hiding the output (message #35) answers the undercount in message #23 but not the demand question. Message #46 reports that worker is now the default and that some providers offer only workers. No reply from Andres or Tom follows v12 or v13.

Thread: [#17 (Andres Freund, 2025-02-09)](https://www.postgresql.org/message-id/myw76agwrlmisvamwbfl6ibxgwh5glzitydiwnfmtb5aui232i%40274yxxtnbnsp), [#23 (Andres Freund, 2025-02-10)](https://www.postgresql.org/message-id/mzjytadtjqv4knvwukppaol5zx2qzt2bkuqvjlo4rjawvhn4ql%40qmdf64qkmxjo), [#26 (Andres Freund, 2025-02-11)](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#35 (torikoshia, 2025-04-11)](https://www.postgresql.org/message-id/0310f2ff46d990485f96624ac1744c01%40oss.nttdata.com), [#46 (Lukas Fittl, 2026-05-25)](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com), [#49 (torikoshia, 2026-08-19)](https://www.postgresql.org/message-id/95ff66682bec0d10e18ea9c6a3dd3f93%40oss.nttdata.com)

**Question:** What share of Postgres 19 and later installations can use sync or io_uring, and do the AIO maintainers accept hiding the output under worker?

Counter-check: held. Message #25 proposed hiding the numbers under worker, and Andres answered in message #26 that the feature then fails in the expected default. Nothing after v12 shows that he or Tom accepts the hiding approach.

### 3. An extension could add this through existing EXPLAIN and executor hooks

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), ABI / extension API, maintenance burden ("who maintains this in 10 years") · `already-exists-2`

Core takes on a counter that is empty under the default io_method = worker, always zero on Windows, and counted in different units on each platform. Andres expects to receive the resulting bug reports (message #26). An extension owner could carry the same cost alone, as pg_stat_kcache has done for years.

pg_stat_kcache already reads these counters and splits shared-buffer hits, OS-cache hits and disk reads per query (messages #3, #11). Extensions can already wrap the executor, and recent releases let them add EXPLAIN options and output. The only piece that may need core is passing worker counts to the leader. A small hook for that would cost less than a core option whose behaviour depends on io_method.

Thread: [#3 (torikoshia, 2024-12-26)](https://www.postgresql.org/message-id/6023c8ac6b1823e2126a001f668ef245%40oss.nttdata.com), [#11 (Julien Rouhaud, 2025-01-06)](https://www.postgresql.org/message-id/Z3upZkA5Bi6lYS6a%40jrouhaud), [#26 (Andres Freund, 2025-02-11)](https://www.postgresql.org/message-id/3gz6ymrgnua75aviagsl4d4traoqxo2g2rzzykqa3yl4jyts3y%40gj6lcc6aziil), [#46 (Lukas Fittl, 2026-05-25)](https://www.postgresql.org/message-id/CAP53PkyM8uRn9uPrEAmv6UaO%2B-O0f5VtQFGDWGWC3BD-Zo1UWQ%40mail.gmail.com)

**Question:** Which part of the feature, other than parallel-worker aggregation, cannot be built in an extension with today's executor and EXPLAIN extension hooks?

Counter-check: held. Extensions can register EXPLAIN options and wrap planner and executor, and pg_stat_kcache reads the same counters. Messages #4, #7 and #46 show interest in core support, but the thread does not weigh an extension plus a small parallel-worker hook.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes instrument.h and execParallel.h to pass storage I/O counts from parallel workers to the leader. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | EXPLAIN (ANALYZE, IO) gains "Storage I/O" lines in text output and two new properties in JSON, XML and YAML. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Output depends on io_method and on getrusage() semantics per platform, and the patch changes the Windows getrusage() port. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | pg_stat_kcache already reads the same getrusage() counters, and extensions can add EXPLAIN options. |

## 5. Questions the author must answer before anyone reviews code

1. Do the AIO maintainers accept hiding Storage I/O output under io_method = worker, given the objection in message #26?
2. What share of Postgres 19 and later installations can use io_method = sync or io_uring, and what evidence shows that share?
3. Which part of the feature, other than passing parallel-worker counts to the leader, cannot be built as an extension EXPLAIN option?
4. How does a text reader or a JSON consumer tell "not measured" (Windows, io_method = worker) apart from a measured zero?
5. Which unit and counting rule do ru_inblock and ru_oublock follow on each supported platform other than Linux?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund and Tom Lane took part, but as objectors, and neither has replied since v12. The owner would need to know the AIO subsystem and EXPLAIN instrumentation.
- **Liability:** The owner signs for an EXPLAIN output whose correctness depends on which process does the I/O, and for getrusage() semantics per platform. Each later change to io_method or to where backend I/O runs must recheck the gating.
- **Author follow-up:** The author has kept the patch rebased through v13 across 9 CommitFests and plans an auto_explain follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Only io_method = sync and io_uring users see the output, and the thread gives no numbers for their share. The 75 KiB patch touches EXPLAIN, parallel executor instrumentation, parallel index builds and the Windows port, and the hiding approach has no visible agreement from the AIO maintainers.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Storage I/O counts mean different things on different platforms. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Absent output, zero and real zero cannot be told apart. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Correct output depends on gating that future I/O changes must remember. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-3` IO option gets a second, different meaning. Refuted (no failure scenario): The counts appear in the separate Planning and Execution groups at the end of the plan, and the documentation says they cover the whole query, unlike the per-node data. The subtraction scenario needs a reader to ignore both.
- `maintenance-fit-3` Extension path for this metric is not weighed against core. Refuted (not specific): Duplicate of already-exists-2, which states the extension path more exactly.
- `already-exists-1` log_executor_stats already reports the same getrusage block counters. Refuted (already handled): log_executor_stats needs superuser-level SET, omits parallel workers, and prints an undercount under io_method = worker with no gating. So it cannot do the job for parallel queries, where disk-bound scans are common, and two reporting paths that differ is not a failure.
- `demand-1` Feature reaches only sync and io_uring users, a shrinking group. Refuted (not specific): Duplicate of maintenance-fit-1.
- `demand-2` The stated main use, auto_explain, is deferred. Refuted (no failure scenario): Messages #9 and #20 show manual EXPLAIN runs that separate cached reads from storage reads and expose hint-bit writes. auto_explain support builds on the EXPLAIN part, so delivering EXPLAIN first is the usual order and not a gap in the use case.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches this idea. [P9] reverts planning-phase BUFFERS output after a buildfarm failure, and its recorded reason does not apply to this proposal. The in-thread PAGEFAULTS version (message #1) was dropped for ru_inblock and ru_oublock after messages #8 to #10.
- standards-compat: 3 objection(s). Other angles, such as worker-based AIO accounting and demand, belong to other panel members.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state: no on-disk format, WAL record, catalog, wire protocol or replication change. Its only cross-process state is per-query parallel-worker shared memory, freed with the query. The edits to nbtsort.c, brin.c, gininsert.c and vacuumparallel.c appear to pass parallel instrumentation only. The nbtree VACUUM bookkeeping invariant does not apply. The instrument.h and execParallel.h struct changes target a major release, where ABI breaks are accepted.
- security: 0 objection(s). The proposal adds no new privilege surface that I can find. EXPLAIN (ANALYZE, IO) runs the query with the caller's own privileges, and getrusage() reports only that backend's counters and its parallel workers' counters, so a user sees nothing from other sessions or relations they cannot read. The counts show whether the caller's own reads hit the OS page cache, which query timing already shows. No new GUC, role or default changes. Without the code I could not check whether worker counters can mix with another query's.
- maintenance-fit: 3 objection(s). Cost for non-users looks contained: getrusage() and the parallel shared memory are used only when IO is requested (message #49). The nbtree invariant on VACUUM bookkeeping does not apply, because the nbtsort.c change only adds parallel instrumentation.
- already-exists: 2 objection(s). Two objections from the already-exists angle. The io_method = worker coverage gap and the bgwriter/checkpointer attribution issue belong to other angles.
- demand: 2 objection(s). The thread shows real interest in telling OS cache hits apart from storage reads (messages #5, #7, #11, #46). The objections above concern how many users the design as proposed can reach.

### Precedents retrieved

- P1: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P2: [explain_regress, explain(MACHINE), and default to explain(BUFFERS)](https://commitfest.postgresql.org/patch/3409/) (cf, Withdrawn, 2023-01-15) - judged not relevant; reason summarised from its thread
- P3: [Connection string usage for Core Postgresql client applications](https://commitfest.postgresql.org/patch/2354/) (cf, Withdrawn, 2020-01-21) - judged not relevant; reason summarised from its thread
- P4: [pg_regress.c: Fix "make check" on Mac OS X: Pass DYLD_LIBRARY_PATH](https://commitfest.postgresql.org/patch/4340/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P5: [Revert change of inclusion order for pg_config_os.h;](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5e04f3857c9369351edbf8f906d1d8c2665c8a1f) (revert, Reverted, 2003-05-15) - judged not relevant
- P6: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P7: [Support run-time partition pruning for hash join](https://commitfest.postgresql.org/patch/4512/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [Run-time pruning for ModifyTable](https://commitfest.postgresql.org/patch/2173/) (cf, Returned with feedback, 2020-04-07) - judged not relevant; reason summarised from its thread
- P9: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - used in the review
- P10: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant; reason summarised from its thread
- P11: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P12: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P13: [Adaptive query optimization: create extented statistics using auto_explain extension](https://commitfest.postgresql.org/patch/2958/) (cf, Returned with feedback, 2021-11-04) - judged not relevant
- P14: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant
- P15: [More stable query plans via more predictable column statistics](https://commitfest.postgresql.org/patch/434/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P16: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P17: [Revert "Disallow infinite endpoints in generate_series() for timestamps."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29904f5f2fdafbbb96ef3685fd361053b061aeb1) (revert, Reverted, 2022-05-09) - judged not relevant
- P18: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P19: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P20: [Logging plan of the currently running query](https://commitfest.postgresql.org/patch/3142/) (cf, Withdrawn, 2023-06-06) - judged not relevant
- P21: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P23: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P24: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 21; wall time: 188 s; cost reported by the CLI: $4.93
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 106 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:32:24.949Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
