# Design pre-review: Progress reporting: a debug trace and a test framework

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7331](https://commitfest.postgresql.org/patch/7331/); [thread](https://www.postgresql.org/message-id/flat/179129532538.231424.8584066696703033814%40gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: ten progress fixes this year and a live bug in WaitForOlderSnapshots() show the area needs tests. The surviving objections are fixable design points at level 2. maintenance-fit-1 and already-exists-1 ask where the checks would run, since default builds skip most of test_progress. maintenance-fit-2 asks how much of the test is tied to execution order. No level fails outright, but only the author has spoken in the visible thread.

Not checked: Without the code, nobody can check that the trace is safe inside critical sections or that default builds are unchanged beyond the author's object-code comparison. The share of ProgressCheck.pm that is tied to execution order is also not visible.

## Prior attempts

None found before 2026-10-06.

## Objections a reviewer is likely to raise

### 1. Most checks run only on builds that define PROGRESS_DEBUG

A committer changes the phases of VACUUM or REPACK and runs cfbot and the buildfarm, and everything passes because test_progress skips there. The expected sequences in ProgressCheck.pm go stale. The next developer who builds with PROGRESS_DEBUG finds failures that someone else's commit caused, possibly months later.

The commit message says that without PROGRESS_DEBUG only the progress.h description check runs, and measurements.txt shows test_progress taking 0.36 s on default builds. No part of the proposal names a buildfarm animal or cfbot task that defines the macro. A check that fails only on builds nobody routinely runs does not warn the person who broke it. The author keeps the compile option because of Álvaro's view in message #2, but trace_progress already keeps the trace out of other tests' logs.

**Question:** Which cfbot task or buildfarm animal will build with PROGRESS_DEBUG, or why can the code not always be compiled in and gated by trace_progress alone?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com); `maintenance-fit-1`</sub>

### 2. New compile macro and GUC where injection points could carry the trace

No buildfarm animal or cfbot task defines PROGRESS_DEBUG, so on every build in the cycle test_progress runs only the progress.h description check and skips the rest. A later patch breaks a phase sequence, and nobody finds out until someone builds with the macro by hand.

Postgres already has a build option for test-only hooks in backend code: --enable-injection-points, and cfbot and some buildfarm animals already use it. An injection point in backend_progress.c, with a callback in the test module that logs the backend's progress state, would give the same trace without a new macro in pg_config_manual.h or a new GUC. Message #1 and the commit message name injection points only as a way to sample the views at one moment, not as a way to carry the trace. Following trace_syncscan is existing practice, but that practice is rarely compiled in.

**Question:** Why can an injection point in backend_progress.c, with a callback in test_progress, not produce the same trace on builds that already enable injection points?

<sub>fixable, level 2, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com); `already-exists-1`</sub>

### 3. Exact per-command sequences make every progress change edit the test module

A patch adds a VACUUM index cycle or reorders a REPACK step and changes no progress semantics. 001_progress.pl then fails on the exact phase succession or on a value checked per parameter. The patch author must learn and edit a large Perl replay module that no committer in the visible thread has said they will look after.

v6-0001 is 118 KB, and its commit message says the test checks the succession of phases and an expected value for every parameter of every command, with failure variants, parallel builds, autovacuum and failsafe. This couples test expectations to the internal order of execution of many commands, beyond the documented contract. Only the author's messages appear. Michael and Álvaro are mentioned in message #2 but have not replied, so no qualified committer has taken on the aftermath. The @ACCEPTED list also needs someone to keep it current.

**Question:** Which checks follow from documented progress semantics and which from current execution order, and which committer has agreed to keep ProgressCheck.pm current when commands change?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/179129532538.231424.8584066696703033814%40gmail.com), [#2](https://www.postgresql.org/message-id/179381939119.1041924.8156751512243472203%40gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Which cfbot task or buildfarm animal will build with PROGRESS_DEBUG, and has its owner agreed?
2. Why can the trace code not always be compiled in and gated by trace_progress alone?
3. Why can an injection point in backend_progress.c, with a callback in test_progress, not produce the same trace on builds that already enable injection points?
4. Which checks in 001_progress.pl follow from documented progress semantics, and which only from the current order of execution?
5. How does the v6 trace build its log line when progress is reported inside a critical section?

## Touch points

- **new GUC:** The patch adds the developer option trace_progress in guc_parameters.dat, present only in PROGRESS_DEBUG builds.
- **maintenance burden ("who maintains this in 10 years"):** ProgressCheck.pm holds a description of every progress parameter and exact phase sequences that each progress change must update.
- **performance on the hot path:** The patch edits backend_progress.c and backend_status.c; the author reports an unchanged instruction stream without PROGRESS_DEBUG.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A new compile macro carries the trace although injection points are an existing build option for test hooks.

## Who would commit it, and when

- Owner: silent. A committer who knows backend_progress.c and the TAP test infrastructure would have to own it. The author mentions Michael and Álvaro, but neither has posted in the visible thread.
- Priority: needs champion; audience niche; review cost high. The audience is Postgres developers who change progress reporting, and the trace has already found bugs on master. v6-0001 is 118 KB, mostly Perl expectations for many commands, and no committer has replied in the visible thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
