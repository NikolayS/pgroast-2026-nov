# Design pre-review: Progress reporting: a debug trace and a test framework

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7331](https://commitfest.postgresql.org/patch/7331/) (Needs review); authors: Manuel Reyes Bravo (manu15)
- Thread: [Progress reporting: a debug trace and a test framework](https://www.postgresql.org/message-id/flat/179129532538.231424.8584066696703033814%40gmail.com), started 2026-10-06
- Messages read: 2 of 2
- Patch prose read: v6-0001-Add-PROGRESS_DEBUG-to-test-the-whole-sequence-of-.patch, v6-0002-Check-the-documented-progress-phases-against-the-.patch, measurements.txt, nocfbot-mutation-benchmark.txt, repro-waitfor.sh.txt, v5-0001-Add-PROGRESS_DEBUG-to-test-the-whole-sequence-of-.patch, v5-0002-Check-the-documented-progress-phases-against-the-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: ten progress fixes this year and a live bug in WaitForOlderSnapshots() show the area needs tests. The surviving objections are fixable design points at level 2. maintenance-fit-1 and already-exists-1 ask where the checks would run, since default builds skip most of test_progress. maintenance-fit-2 asks how much of the test is tied to execution order. No level fails outright, but only the author has spoken in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check that the trace is safe inside critical sections or that default builds are unchanged beyond the author's object-code comparison. The share of ProgressCheck.pm that is tied to execution order is also not visible.

## The proposal, restated

**Problem.** Progress reporting is tested only by sampling pg_stat_progress_* views at one moment, which needs concurrency or injection points and misses the succession of values a command reports. Bugs in progress values and undocumented phases or parameters go unnoticed. The audience is Postgres developers and the regression suite.

**Proposed design.** A compile-time macro PROGRESS_DEBUG, plus a developer GUC trace_progress, logs every progress state change at LOG_SERVER_ONLY from backend_progress.c. A new TAP module test_progress replays each backend's trace and checks general rules and per-command phase and value sequences. A second test checks that reported phases appear in system_views.sql and monitoring.sgml.

**User-visible surface.**
- Compile-time macro PROGRESS_DEBUG in pg_config_manual.h.
- Developer GUC trace_progress, modeled on trace_syncscan, present only in PROGRESS_DEBUG builds.
- Server log lines "progress start/update/end: <command> relid=... idx:old->new" when the trace is on.
- New test module src/test/modules/test_progress with ProgressCheck.pm and DocPhases.pm.

**What the author claims.**
- Without PROGRESS_DEBUG the object code has the same instruction stream as master, and meson test time differs within noise (84.9 s against 84.2 s).
- With the option, meson test takes about 7 s more and check-world logs grow from about 70 MB to 135 MB, all from test_progress's own trace.
- Reverting this year's ten progress fixes one at a time, v6 catches nine; five of the checks were written knowing those fixes.
- Of 301 held-out blind mutants of progress calls, test_progress catches 285 (94.7%); today's tests catch 10 of a 150-mutant sample.
- The trace found a live bug: REINDEX CONCURRENTLY shows lockers_done 2 with lockers_total 1, because WaitForOlderSnapshots() does not reset PROGRESS_WAITFOR_* values.

**Thread state.** Only the author's two messages are visible. He mentions that Michael suggested a separate thread and that Álvaro thought a compile option was enough, but no replies from others appear.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-06; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Most checks run only on builds that define PROGRESS_DEBUG

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer changes the phases of VACUUM or REPACK and runs cfbot and the buildfarm, and everything passes because test_progress skips there. The expected sequences in ProgressCheck.pm go stale. The next developer who builds with PROGRESS_DEBUG finds failures that someone else's commit caused, possibly months later.

The commit message says that without PROGRESS_DEBUG only the progress.h description check runs, and measurements.txt shows test_progress taking 0.36 s on default builds. No part of the proposal names a buildfarm animal or cfbot task that defines the macro. A check that fails only on builds nobody routinely runs does not warn the person who broke it. The author keeps the compile option because of Álvaro's view in message #2, but trace_progress already keeps the trace out of other tests' logs.

Thread: [#1 (Manu, 2026-10-06)](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2 (Manu, 2026-11-04)](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com)

**Question:** Which cfbot task or buildfarm animal will build with PROGRESS_DEBUG, or why can the code not always be compiled in and gated by trace_progress alone?

Counter-check: held. The commit message says the rest of the test is skipped without PROGRESS_DEBUG, and neither message names a cfbot task or buildfarm animal that defines it. Following trace_syncscan explains the macro but does not say where the checks run.

### 2. New compile macro and GUC where injection points could carry the trace

fixable · level 2, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

No buildfarm animal or cfbot task defines PROGRESS_DEBUG, so on every build in the cycle test_progress runs only the progress.h description check and skips the rest. A later patch breaks a phase sequence, and nobody finds out until someone builds with the macro by hand.

Postgres already has a build option for test-only hooks in backend code: --enable-injection-points, and cfbot and some buildfarm animals already use it. An injection point in backend_progress.c, with a callback in the test module that logs the backend's progress state, would give the same trace without a new macro in pg_config_manual.h or a new GUC. Message #1 and the commit message name injection points only as a way to sample the views at one moment, not as a way to carry the trace. Following trace_syncscan is existing practice, but that practice is rarely compiled in.

Thread: [#1 (Manu, 2026-10-06)](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2 (Manu, 2026-11-04)](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com)

**Question:** Why can an injection point in backend_progress.c, with a callback in test_progress, not produce the same trace on builds that already enable injection points?

Counter-check: held. The thread mentions injection points only for sampling the views, and does not say why one in backend_progress.c could not carry the trace. Copying trace_syncscan lowers the weight, but the question stands because cfbot already builds with injection points.

### 3. Exact per-command sequences make every progress change edit the test module

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A patch adds a VACUUM index cycle or reorders a REPACK step and changes no progress semantics. 001_progress.pl then fails on the exact phase succession or on a value checked per parameter. The patch author must learn and edit a large Perl replay module that no committer in the visible thread has said they will look after.

v6-0001 is 118 KB, and its commit message says the test checks the succession of phases and an expected value for every parameter of every command, with failure variants, parallel builds, autovacuum and failsafe. This couples test expectations to the internal order of execution of many commands, beyond the documented contract. Only the author's messages appear. Michael and Álvaro are mentioned in message #2 but have not replied, so no qualified committer has taken on the aftermath. The @ACCEPTED list also needs someone to keep it current.

Thread: [#1 (Manu, 2026-10-06)](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2 (Manu, 2026-11-04)](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com)

**Question:** Which checks follow from documented progress semantics and which from current execution order, and which committer has agreed to keep ProgressCheck.pm current when commands change?

Counter-check: held. The commit message confirms exact phase successions and a value per parameter for every command, so changes to execution order will need edits in ProgressCheck.pm. The missing committer owner belongs under committability and should not add weight to the verdict.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds the developer option trace_progress in guc_parameters.dat, present only in PROGRESS_DEBUG builds. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | ProgressCheck.pm holds a description of every progress parameter and exact phase sequences that each progress change must update. |
| performance on the hot path | **yes** | The patch edits backend_progress.c and backend_status.c; the author reports an unchanged instruction stream without PROGRESS_DEBUG. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A new compile macro carries the trace although injection points are an existing build option for test hooks. |

## 5. Questions the author must answer before anyone reviews code

1. Which cfbot task or buildfarm animal will build with PROGRESS_DEBUG, and has its owner agreed?
2. Why can the trace code not always be compiled in and gated by trace_progress alone?
3. Why can an injection point in backend_progress.c, with a callback in test_progress, not produce the same trace on builds that already enable injection points?
4. Which checks in 001_progress.pl follow from documented progress semantics, and which only from the current order of execution?
5. How does the v6 trace build its log line when progress is reported inside a critical section?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows backend_progress.c and the TAP test infrastructure would have to own it. The author mentions Michael and Álvaro, but neither has posted in the visible thread.
- **Liability:** The owner signs for a 118 KB test module whose per-command expectations need an edit whenever a command's progress reporting or execution order changes, for as long as the module exists. The owner also signs for the trace code in backend_progress.c, the @ACCEPTED list and the PROGRESS_DEBUG build staying green.
- **Author follow-up:** The author posted v6 four weeks after v5 with measurements, a mutation benchmark and a bug reproduction, and offered a follow-up patch for the counter resets.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The audience is Postgres developers who change progress reporting, and the trace has already found bugs on master. v6-0001 is 118 KB, mostly Perl expectations for many commands, and no committer has replied in the visible thread.

## Appendix

### Refuted in the counter-check

- `demand-1` Most checks run only in builds nobody is shown to make. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same gap and also names the alternative of gating by trace_progress alone.
- `demand-2` Every progress patch must update ProgressCheck.pm, for a few users. Refuted (wrong facts): The description check runs on any build, so its gain is not limited to PROGRESS_DEBUG builds; message #1 shows it flagging the undescribed parameter from 1378aa13430. The per-command value expectations are skipped on default builds; maintenance-fit-2 covers that cost.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. P1 and P2 add a checkpoint progress view and were returned for view complexity, unused fields and checkpoint overhead, none of which applies to a compiled-out trace and test module. Message #1 says the patches were first posted in another thread and moved here at Michael's suggestion, and message #2 reports that Álvaro thought a compile option was enough. The visible thread records no rejection reason from that earlier thread.
- standards-compat: 0 objection(s). The proposal adds no SQL syntax, catalog object, dumped setting or psql change. The trace_progress developer option follows the existing TRACE_SYNCSCAN/trace_syncscan pattern and exists only in PROGRESS_DEBUG builds, so default builds and their users see no change. The listed files do not include config.sgml, so the option's entry under developer options is not shown. The proposal does not say whether the log line format is meant to stay stable for tools; both points are documentation details, not standards objections.
- replication-ondisk: 0 objection(s). The proposal adds no on-disk, WAL, catalog, replication or wire-protocol state. It is a compile-time trace and a developer GUC that write server log lines, plus a TAP module. Default builds keep the same instruction stream, so extension ABI is unchanged; logging inside critical sections in PROGRESS_DEBUG builds can only be checked in the code.
- security: 0 objection(s). The trace is compiled in only with PROGRESS_DEBUG, a developer build option, and writes only relids and integer progress values to the server log at LOG_SERVER_ONLY, never to clients. Even if trace_progress were user-settable like trace_syncscan, the worst case is log growth on a debug build, and production builds are not affected.
- maintenance-fit: 2 objection(s). Hot-path cost is not raised: the trace is compiled out by default, and measurements.txt shows the same instruction stream and test time within noise. The listed precedents concern a checkpoint progress view and do not apply.
- already-exists: 1 objection(s). The views and injection points only sample values at one moment, so they do not cover the full sequence of reports, and no extension can see progress writes. The objection above is about reusing an existing mechanism, and the need itself is real.
- demand: 2 objection(s). The problem itself is shown. Ten progress fixes this year, a live WaitForOlderSnapshots() bug, and current tests catching 10 of 150 mutants show the area needs testing. The open demand questions are who runs the PROGRESS_DEBUG build and who pays for maintaining the descriptions.

### Precedents retrieved

- P1: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P3: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P4: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant; reason summarised from its thread
- P5: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P6: [improve DEBUG1 logging of parallel workers for CREATE INDEX](https://commitfest.postgresql.org/patch/5490/) (cf, Withdrawn, 2025-02-03) - judged not relevant; reason summarised from its thread
- P7: [Enhance file_fdw to report processed and skipped tuples in COPY progress](https://commitfest.postgresql.org/patch/5282/) (cf, Withdrawn, 2025-03-09) - judged not relevant; reason summarised from its thread
- P8: [Add description about using geqo to EXPLAIN or debug log.](https://commitfest.postgresql.org/patch/3680/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P9: [Progress reporting for COPY command](https://commitfest.postgresql.org/patch/2607/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P10: [logical streaming for large in-progress transactions](https://commitfest.postgresql.org/patch/1429/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P11: [Revert setting client_min_messages to 'debug1' in new tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5655565c077c53b6e9b4b9bfcdf96439cf3af065) (revert, Reverted, 2019-03-13) - judged not relevant
- P12: [VACUUM Progress Checker](https://commitfest.postgresql.org/patch/349/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P13: [Revert removal of trace_userlocks, because userlocks aren't gone.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71b2b657c034743f81ae7906d7b0fcc8e9a0a6a3) (revert, Reverted, 2011-11-10) - judged not relevant
- P14: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P15: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P16: [Refactor compile-time assertion checks for C/C++](https://commitfest.postgresql.org/patch/2437/) (cf, Returned with feedback, 2020-03-23) - judged not relevant
- P17: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant
- P18: [Revert "Temporarily install debugging in partition_prune test"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec1a405adc24bc20523eef83ccc85232ec23eaa9) (revert, Reverted, 2024-05-14) - judged not relevant
- P19: [Revert "Refactor compile-time assertion checks in c.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa650a99b4bfbe95e88f44070ae0df1818944a7) (revert, Reverted, 2020-03-23) - judged not relevant
- P20: [pg_waldump: add test for coverage](https://commitfest.postgresql.org/patch/3834/) (cf, Returned with feedback, 2022-09-30) - judged not relevant
- P21: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P22: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant
- P23: [Generate pkg-config for server module development](https://commitfest.postgresql.org/patch/6367/) (cf, Withdrawn, 2026-03-27) - judged not relevant
- P24: [pg_checksum: add test for coverage](https://commitfest.postgresql.org/patch/3852/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 116 s; cost reported by the CLI: $2.03
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 72 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:41:31.485Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
