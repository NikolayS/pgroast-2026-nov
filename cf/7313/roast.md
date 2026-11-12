# Design pre-review: Row contention in regress/parallel_schedule, adding multi-line test groups

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7313](https://commitfest.postgresql.org/patch/7313/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxObF4iYtNA-3w%2BJDNT2YsbS5YMyGiPMnYU0Gd0CtUOMGQ%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and two reviewers support multi-line groups. Three fixable design points remain. The indentation grammar is not reconciled with Daniel's looser rule or the pending scheduling work (standards-compat-2). The rewritten parallel_schedule exists only on master (maintenance-fit-1). The concurrency output is bundled without a named consumer (maintenance-fit-3). The reviewers already in the thread can settle all three.

Not checked: Whether the parser implements the grammar in the commit message could not be checked; the author says they may differ and lists no parser tests. Messages #4 to #6 were judged only through the objections and the brief.

## Prior attempts

None found before 2026-09-17.

## Objections a reviewer is likely to raise

### 1. Indentation-based grammar fixed now, while a known redesign is pending

Extension authors running pg_regress --schedule adopt indented continuation lines in their own schedule files. The scheduling work in message #4 then needs a different syntax and must either keep the indentation rule forever or break those out-of-tree schedules.

In message #4 Daniel reports unpublished scheduling work that will also change the syntax, and he suggests a whitespace-insensitive rule: any names between ^test: and the next test: or EOF. The v2 grammar instead makes indentation meaningful and makes every unindented line an error. Message #5 sketches a third syntax ("test sequentially:", "test concurrently:") and asks to settle multi-line first. That gives two syntax changes in a row with no agreement on how they fit together.

**Question:** How does the indentation rule fit with the pending dependency-aware scheduling syntax, and why is it better than Daniel's whitespace-insensitive rule?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/41CF553C-6303-4CC6-B114-85B4011799C2%40yesql.se), [#5](https://www.postgresql.org/message-id/CAE8JnxPycpY5mk7C-rNa4jeMQQXG1H5mobB10Oa_CaQs1nBvUg%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Rewriting parallel_schedule breaks clean backpatching of test additions

A committer fixes a bug, adds a regression test to parallel_schedule on master and backpatches it to all supported branches. Master uses one test per line and the back branches use long single lines, so the schedule hunk fails to apply on every branch. This repeats for each backpatched schedule change until the last pre-change branch goes out of support.

The patch converts all of src/test/regress/parallel_schedule, and the thread says nothing about the stable branches. The rebase conflicts it removes are between pending master patches. The conflicts it adds fall on committers doing backpatches, for about five years. The thread does not weigh one cost against the other.

**Question:** Will the parser change be backpatched, or how often do backpatched fixes touch parallel_schedule, and is that cost smaller than the rebase cost saved on master?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAE8JnxObF4iYtNA-3w%2BJDNT2YsbS5YMyGiPMnYU0Gd0CtUOMGQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAE8JnxOCb%2BO1k-%2B%3D2vnMLPLo-ZrsyC--%2BiKZWLFopOh9DQQLAw%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Concurrency logging is a separate feature bundled with the syntax

A committer who agrees with the syntax change must also accept a new output line for every parallel group in every regression run, buildfarm log and CI log. If the number turns out noisy or misleading on loaded machines, removing it later means touching the same patch that changed the schedule format.

v2 adds the "effective concurrency" output only as a first step toward faster scheduling, with no consumer yet (message #5). It is unrelated to rebase conflicts and was not requested by either reviewer. As one combined commit, a committer can only take both parts or neither.

**Question:** Can the concurrency output be split into its own patch, and what decision or tool will use this number?

<sub>fixable, level 2, design; messages [#5](https://www.postgresql.org/message-id/CAE8JnxPycpY5mk7C-rNa4jeMQQXG1H5mobB10Oa_CaQs1nBvUg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAE8JnxOCb%2BO1k-%2B%3D2vnMLPLo-ZrsyC--%2BiKZWLFopOh9DQQLAw%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Why does v2 make indentation meaningful and unindented lines an error, when message #4 suggests taking any names between one "test:" and the next?
2. Does the v2 grammar fit the scheduling work mentioned in message #4 and the "test sequentially:" form in message #5, so converted schedules need no second rewrite?
3. Will the parser change be back-patched, and if not, how do committers apply parallel_schedule edits to stable branches?
4. Can the effective concurrency output become a separate patch, and which tool or decision will use the number?
5. Does any schedule file that pg_regress accepts today fail under v2, and is the schedule syntax documented anywhere that needs an update?

## Touch points

- **ABI / extension API:** Out-of-tree users of pg_regress write schedule files, and v2 makes every unindented non-"test:" line an error.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: Indentation-based grammar fixed now, while a known redesign is pending
- **backpatchability:** The patch rewrites parallel_schedule on master only, so back-patched schedule edits will not apply to stable branches.
- **maintenance burden ("who maintains this in 10 years"):** pg_regress gains a second schedule form and a new output line per parallel group, with another syntax change pending.

## Who would commit it, and when

- Owner: qualified committer engaged. Someone who knows pg_regress and the regression test schedules. Michael Paquier and Daniel Gustafsson both reviewed in the thread, and Daniel has earlier pg_regress work (P2).
- Priority: proceed; audience niche; review cost low. The change helps Postgres developers who edit parallel_schedule and touches only pg_regress.c and the schedule file. Two reviewers are engaged, and the open points are a grammar choice, a back-patch policy and a patch split.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
