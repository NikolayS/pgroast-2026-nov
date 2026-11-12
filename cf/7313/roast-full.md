# Design pre-review: Row contention in regress/parallel_schedule, adding multi-line test groups

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7313](https://commitfest.postgresql.org/patch/7313/) (Needs review); authors: Alexandre Felipe (afelipe)
- Thread: [pg_regress: schedule multi-line test groups](https://www.postgresql.org/message-id/flat/CAE8JnxObF4iYtNA-3w%2BJDNT2YsbS5YMyGiPMnYU0Gd0CtUOMGQ%40mail.gmail.com), started 2026-09-17
- Messages read: 6 of 6
- Patch prose read: v2.1-0001-pg_regress-multi-line-test-groups.patch, v1-0001-pg_regress-multi-line-test-groups.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and two reviewers support multi-line groups. Three fixable design points remain. The indentation grammar is not reconciled with Daniel's looser rule or the pending scheduling work (standards-compat-2). The rewritten parallel_schedule exists only on master (maintenance-fit-1). The concurrency output is bundled without a named consumer (maintenance-fit-3). The reviewers already in the thread can settle all three.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Whether the parser implements the grammar in the commit message could not be checked; the author says they may differ and lists no parser tests. Messages #4 to #6 were judged only through the objections and the brief.

## The proposal, restated

**Problem.** Every change to parallel_schedule edits long single lines, so unrelated patches conflict and need rebases; the author hit two in one week on another patch. The author also finds long lines hard to read and wants one test per line, so line numbers show group size. The audience is Postgres developers.

**Proposed design.** pg_regress accepts a group that starts with "test:" and continues on following indented lines until the next unindented line. Blank lines and comments may appear inside a group. Unindented non-"test:" lines and empty groups are errors, and v2 also logs each parallel group's measured average concurrency.

**User-visible surface.**
- Schedule file syntax: "test:" may be followed by tests on indented continuation lines, mixed with tests on the "test:" line.
- New pg_regress error messages for unindented lines, naming the line, and for empty test groups.
- New pg_regress output line per group: "# effective concurrency 1.46 / 2".
- src/test/regress/parallel_schedule rewritten to the multi-line form.

**What the author claims.**
- Multi-line groups reduce rebase conflicts in parallel_schedule.
- One test per line is easier to read and makes test counts visible from line numbers.
- Logging effective concurrency is a first step toward faster scheduling; "we can't optimise what we don't measure".
- Existing single-line schedules keep working.

**Thread state.** Michael Paquier gave a +1 and asked for consistent whitespace; Daniel Gustafsson found that v1 accepted empty groups, which v2 fixes, and mentioned unpublished scheduling work that will also change the syntax. Nobody has reviewed v2.1 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-17; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Indentation-based grammar fixed now, while a known redesign is pending

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Extension authors running pg_regress --schedule adopt indented continuation lines in their own schedule files. The scheduling work in message #4 then needs a different syntax and must either keep the indentation rule forever or break those out-of-tree schedules.

In message #4 Daniel reports unpublished scheduling work that will also change the syntax, and he suggests a whitespace-insensitive rule: any names between ^test: and the next test: or EOF. The v2 grammar instead makes indentation meaningful and makes every unindented line an error. Message #5 sketches a third syntax ("test sequentially:", "test concurrently:") and asks to settle multi-line first. That gives two syntax changes in a row with no agreement on how they fit together.

Thread: [#4 (Daniel Gustafsson, 2026-09-18)](https://www.postgresql.org/message-id/41CF553C-6303-4CC6-B114-85B4011799C2%40yesql.se), [#5 (Alexandre Felipe, 2026-09-20)](https://www.postgresql.org/message-id/CAE8JnxPycpY5mk7C-rNa4jeMQQXG1H5mobB10Oa_CaQs1nBvUg%40mail.gmail.com)

**Question:** How does the indentation rule fit with the pending dependency-aware scheduling syntax, and why is it better than Daniel's whitespace-insensitive rule?

Counter-check: held. Message #4 says pending work will change the syntax, and message #5 asks to land multi-line first without saying how the two fit. The thread also gives no reason for making unindented lines an error instead of using Daniel's looser rule.

### 2. Rewriting parallel_schedule breaks clean backpatching of test additions

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-1`

A committer fixes a bug, adds a regression test to parallel_schedule on master and backpatches it to all supported branches. Master uses one test per line and the back branches use long single lines, so the schedule hunk fails to apply on every branch. This repeats for each backpatched schedule change until the last pre-change branch goes out of support.

The patch converts all of src/test/regress/parallel_schedule, and the thread says nothing about the stable branches. The rebase conflicts it removes are between pending master patches. The conflicts it adds fall on committers doing backpatches, for about five years. The thread does not weigh one cost against the other.

Thread: [#1 (Alexandre Felipe, 2026-09-17)](https://www.postgresql.org/message-id/CAE8JnxObF4iYtNA-3w%2BJDNT2YsbS5YMyGiPMnYU0Gd0CtUOMGQ%40mail.gmail.com), [#6 (Alexandre Felipe, 2026-09-20)](https://www.postgresql.org/message-id/CAE8JnxOCb%2BO1k-%2B%3D2vnMLPLo-ZrsyC--%2BiKZWLFopOh9DQQLAw%40mail.gmail.com)

**Question:** Will the parser change be backpatched, or how often do backpatched fixes touch parallel_schedule, and is that cost smaller than the rebase cost saved on master?

Counter-check: held. The patch rewrites parallel_schedule on master only, and the thread does not mention stable branches. Back-patched fixes that add a new test file may be uncommon, but that is the frequency the objection asks the author to show.

### 3. Concurrency logging is a separate feature bundled with the syntax

fixable · level 2, rubric item 5 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A committer who agrees with the syntax change must also accept a new output line for every parallel group in every regression run, buildfarm log and CI log. If the number turns out noisy or misleading on loaded machines, removing it later means touching the same patch that changed the schedule format.

v2 adds the "effective concurrency" output only as a first step toward faster scheduling, with no consumer yet (message #5). It is unrelated to rebase conflicts and was not requested by either reviewer. As one combined commit, a committer can only take both parts or neither.

Thread: [#5 (Alexandre Felipe, 2026-09-20)](https://www.postgresql.org/message-id/CAE8JnxPycpY5mk7C-rNa4jeMQQXG1H5mobB10Oa_CaQs1nBvUg%40mail.gmail.com), [#6 (Alexandre Felipe, 2026-09-20)](https://www.postgresql.org/message-id/CAE8JnxOCb%2BO1k-%2B%3D2vnMLPLo-ZrsyC--%2BiKZWLFopOh9DQQLAw%40mail.gmail.com)

**Question:** Can the concurrency output be split into its own patch, and what decision or tool will use this number?

Counter-check: held. The commit message lists effective concurrency as a separate section of the same patch, and no reviewer asked for it or commented on it. Nothing in the thread discusses splitting it out or names a consumer for the number.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Out-of-tree users of pg_regress write schedule files, and v2 makes every unindented non-"test:" line an error. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: Indentation-based grammar fixed now, while a known redesign is pending |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The patch rewrites parallel_schedule on master only, so back-patched schedule edits will not apply to stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_regress gains a second schedule form and a new output line per parallel group, with another syntax change pending. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why does v2 make indentation meaningful and unindented lines an error, when message #4 suggests taking any names between one "test:" and the next?
2. Does the v2 grammar fit the scheduling work mentioned in message #4 and the "test sequentially:" form in message #5, so converted schedules need no second rewrite?
3. Will the parser change be back-patched, and if not, how do committers apply parallel_schedule edits to stable branches?
4. Can the effective concurrency output become a separate patch, and which tool or decision will use the number?
5. Does any schedule file that pg_regress accepts today fail under v2, and is the schedule syntax documented anywhere that needs an update?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Someone who knows pg_regress and the regression test schedules. Michael Paquier and Daniel Gustafsson both reviewed in the thread, and Daniel has earlier pg_regress work (P2).
- **Liability:** The owner signs for the schedule grammar, which out-of-tree schedules may adopt. The owner also signs for hand-edited schedule hunks in back-patches until the last single-line branch leaves support, about five years.
- **Author follow-up:** The author posted v1.1 the same day as the first review and v2 and v2.1 after Daniel's empty-group report, so the author responds to feedback.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The change helps Postgres developers who edit parallel_schedule and touches only pg_regress.c and the schedule file. Two reviewers are engaged, and the open points are a grammar choice, a back-patch policy and a patch split.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Rewritten parallel_schedule makes back-patched test additions conflict. Refuted (not specific): Duplicate of maintenance-fit-1. That one states the same back-patch conflict without the unsupported claim that adding a test to a group is the most common bug-fix edit to this file.
- `maintenance-fit-2` Schedule syntax may change twice in a short time. Refuted (not specific): Duplicate of standards-compat-2. This version adds claims the material does not support: that isolation and extension schedules would adopt the form, and that pg_regress would keep three forms.
- `demand-1` Effective concurrency output has no stated user beyond the author. Refuted (not specific): Duplicate of maintenance-fit-3, which already asks for a split and a named consumer. The "about forty lines per run" figure and the scripts that would break are not in the material.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches this proposal. [P18] covers multi-line pgbench scripts and records no reason, so nothing carries over. [P2] covers pg_regress output formats, and its doubt about keeping two formats concerns output, not schedule syntax. In message #4 the thread mentions unpublished scheduling work that would also change the syntax, but no earlier rejection of this idea.
- standards-compat: 2 objection(s). Two objections raised. Existing single-line schedules keep working, and the new concurrency line is a TAP comment, so the output contract is not affected.
- replication-ondisk: 0 objection(s). The proposal changes only the pg_regress schedule file parser, its log output and parallel_schedule. It touches no on-disk format, WAL, replication, catalog, wire protocol or server ABI, so this angle has nothing to object to.
- security: 0 objection(s). The proposal changes the schedule file syntax of pg_regress and adds a timing line to its output, so it has no privilege or exposure aspect. Schedule files are developer-controlled test inputs. Nothing here touches server privileges, authentication, user data or what an unprivileged database user can see or do.
- maintenance-fit: 3 objection(s). The objections are about cost and sequencing. The goal itself has support from Michael Paquier and Daniel Gustafsson.
- already-exists: 0 objection(s). Postgres has no existing way to split one parallel group across lines. Each extra test: line in a schedule file starts a new group that runs after the previous one, so it changes the meaning and cannot replace this. No contrib module or external tool fills the gap, and the schedule format belongs to pg_regress itself, so this cannot live outside core. The effective concurrency line could perhaps be derived from per-test timings, but pg_regress does not print each group's wall time, so I cannot name a working alternative.
- demand: 1 objection(s). Demand for the core change is real: parallel_schedule conflicts affect many patch authors, Michael gave a +1 and Daniel proposed a looser syntax. The only demand gap is the added concurrency output.

### Precedents retrieved

- P1: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P2: [TAP format output for pg_regress](https://commitfest.postgresql.org/patch/3559/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P3: [Detect test suite sideeffects in pg_regress](https://commitfest.postgresql.org/patch/4321/) (cf, Withdrawn, 2024-04-03) - judged not relevant; reason summarised from its thread
- P4: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P5: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant; reason summarised from its thread
- P6: [TAP format output for pg_regress](https://commitfest.postgresql.org/patch/3377/) (cf, Withdrawn, 2021-11-26) - judged not relevant
- P7: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant; reason summarised from its thread
- P8: [Changing the autovacuum launcher scheduling; oldest table first algorithm](https://commitfest.postgresql.org/patch/1573/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P9: [Revert "pg_regress: Don't use absolute paths for the diff"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f27522553992132d34c67caeea193d57934111cf) (revert, Reverted, 2019-02-23) - judged not relevant
- P10: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P11: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P12: [Regression tests for V2 protocol.](https://commitfest.postgresql.org/patch/1303/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P13: [Revert "Tweak "line" test to avoid negative zeros on some platforms"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f7a47912af2206698d14650f2149fa951b8ce07) (revert, Reverted, 2013-10-25) - judged not relevant
- P14: [Revert "Fix broken multibyte regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ad2e233385ee3e441454f20a02ac0f68f7873bc2) (revert, Reverted, 2016-01-25) - judged not relevant
- P15: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P16: [Revert "Stabilize output of new regression test case".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cbadba8dd632fc0d4162f7d686fec631bce7dfd0) (revert, Reverted, 2018-02-14) - judged not relevant
- P17: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P18: [pgbench with multi-line scripts](https://commitfest.postgresql.org/patch/319/) (cf, Returned with feedback, 2015-12-22) - used in the review
- P19: [Revert "Fix isolation test to be less timing-dependent"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c8be5962aea2eee8c366c01bbbcf5bf5ddf5294) (revert, Reverted, 2018-01-03) - judged not relevant
- P20: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/1745/) (cf, Returned with feedback, 2019-06-28) - judged not relevant
- P21: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/2186/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P22: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P23: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P24: [Revert "Stabilize test of BRIN parallel create"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a89cd7bfcdbb45a2c23b4b6004f4503f04f83bd7) (revert, Reverted, 2024-04-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 122 s; cost reported by the CLI: $1.20
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 76 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:40:06.347Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
