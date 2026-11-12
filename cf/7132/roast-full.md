# Design pre-review: meson: Support ECPG tests in runnable cross builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7132](https://commitfest.postgresql.org/patch/7132/) (Needs review); authors: Harrison Booth (hbooth)
- Thread: [\[PATCH v1 0/2\] meson: Support ECPG tests in runnable cross builds](https://www.postgresql.org/message-id/flat/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), started 2026-07-24
- Messages read: 3 of 3
- Patch prose read: 0001-meson-Honor-executable-wrappers-for-regression-tests.patch, 0002-meson-Run-ECPG-tests-in-runnable-cross-builds.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

No objection survived at level 1a or 1b; the goal is clear and the change only turns on existing test coverage. Two fixable level 2 points remain. standards-compat-1 asks whether programs started by pg_regress_ecpg run when a wrapper is required, and maintenance-fit-1 asks what keeps the wrapper path in testwrap tested. A committer who knows the Meson build has to judge both, and none has posted.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, we could not check how testwrap resolves the runner path and builds the wrapper command, or whether the changed test() call alters native builds. Whether pg_regress_ecpg passes the wrapper to the programs it starts also needs the code.

## The proposal, restated

**Problem.** The Meson build skips ECPG test targets in every cross build, even when the build machine can run host binaries. One example is x64 Windows binaries on Windows ARM64 with needs_exe_wrapper=false. Developers who cross-build and want ECPG test coverage are affected.

**Proposed design.** Patch 1 makes the regression test harness honor Meson's executable wrapper. It passes the runner as an executable target to test(), resolves its path before testwrap changes directory, and runs it through MESON_EXE_WRAPPER when set. Patch 2 replaces the cross-build check in the ECPG test meson.build with meson.can_run_host_binaries(). v2 merges both into one patch.

**User-visible surface.**
- ECPG test targets now appear in Meson cross builds when meson.can_run_host_binaries() is true, directly or through an exe_wrapper.
- src/tools/testwrap runs the test runner through MESON_EXE_WRAPPER when that variable is set.

**What the author claims.**
- ECPG passes all 66 subtests (67 in v1) on native ARM64, directly runnable x64, and x64 through a Python wrapper.
- Without a required wrapper, ECPG test targets stay disabled, so the check does not enable a broken path.
- A wrapper in a path with spaces works on GitHub Windows x64 and ARM64 runners with Meson 1.11.2.
- No earlier proposal on the ECPG test guard exists, and a cross file with needs_exe_wrapper=false was suggested in a 2024 thread.

**Thread state.** Only the author has posted, with v1, a rebased v2 and later test results, and he asks for review of the wrapper handling and the ECPG guard. The CommitFest entry moved to PG20-3 and still needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Wrapper covers the runner but not the programs it starts

fixable · level 2, rubric item 13 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A developer cross-builds for aarch64 on x86-64 Linux with exe_wrapper = qemu-aarch64 and no binfmt registration. After the patch the ECPG tests run, and pg_regress_ecpg starts under qemu but likely spawns initdb, postgres and the ECPG test programs without the wrapper. Every test fails, and the build turns red with no configuration change.

Per the commit message, testwrap wraps only the runner, but meson.can_run_host_binaries() also returns true when a wrapper is required. Every wrapper case tested (messages #1 and #3) ran on Windows, where x64 runs natively or through emulation and the Python wrapper only passes calls through. That shows the runner goes through the wrapper, but it does not show that its child processes can run. Linux cross setups with qemu or wine are untested.

Thread: [#1 (Harrison Booth, 2026-07-24)](https://www.postgresql.org/message-id/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), [#2 (harrisontbooth(at)gmail(dot)com, 2026-10-12)](https://www.postgresql.org/message-id/CAAH-eSWwQr0P1p-JvKnEuco0-biwcxfYYx%3D3yUK5UEqCWc9%3D9w%40mail.gmail.com), [#3 (harrisontbooth(at)gmail(dot)com, 2026-10-20)](https://www.postgresql.org/message-id/CAAH-eSUT%3DQaEtzTcDFk7iKEhBsXm-i_6WgfTCjXKF%2BgmmWFH6A%40mail.gmail.com)

**Question:** In a cross build where host binaries run only through the wrapper, how do initdb, postgres and the ECPG test programs started by pg_regress_ecpg get run?

Counter-check: held. Every wrapper run in messages #1 to #3 used a Python wrapper on Windows, where x64 binaries also run without it. No run shows that initdb, postgres and the test programs started by pg_regress_ecpg work when the wrapper is required.

### 2. Wrapper path in testwrap has no recurring test coverage

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch changes testwrap's argument or directory handling, for example the pending EXTRA_REGRESS_OPTS work. Native CI passes because MESON_EXE_WRAPPER is unset there, but the wrapper branch of testwrap is broken. The next person to cross-build with an exe_wrapper finds it months later, with no committer who knows why the branch exists.

Patch 1 adds a second invocation path to src/tools/testwrap, and this path runs only when Meson sets MESON_EXE_WRAPPER. All results in messages #1 to #3 come from the author's own Windows runs, and nothing proposes a CI task or buildfarm animal that sets this path up. Other testwrap changes are in flight, so later edits are likely to reach this branch without anyone running it. No committer has posted in the thread, so ownership of the branch is not settled.

Thread: [#1 (Harrison Booth, 2026-07-24)](https://www.postgresql.org/message-id/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), [#2 (harrisontbooth(at)gmail(dot)com, 2026-10-12)](https://www.postgresql.org/message-id/CAAH-eSWwQr0P1p-JvKnEuco0-biwcxfYYx%3D3yUK5UEqCWc9%3D9w%40mail.gmail.com), [#3 (harrisontbooth(at)gmail(dot)com, 2026-10-20)](https://www.postgresql.org/message-id/CAAH-eSUT%3DQaEtzTcDFk7iKEhBsXm-i_6WgfTCjXKF%2BgmmWFH6A%40mail.gmail.com)

**Question:** Which CI task or buildfarm animal would run regression tests through an exe_wrapper, so a later testwrap change that breaks this path fails before commit?

Counter-check: held. The wrapper branch in testwrap runs only when Meson sets MESON_EXE_WRAPPER, and the thread names no CI task or buildfarm animal that does so. The precedents show other testwrap changes are pending, so a silent break is plausible.

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
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Patch 1 adds a wrapper path to src/tools/testwrap that runs only when Meson sets MESON_EXE_WRAPPER. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. In a cross build where host binaries run only through the wrapper, how do initdb, postgres and the ECPG test programs started by pg_regress_ecpg get run?
2. Which CI task or buildfarm animal would run regression tests through an exe_wrapper, so a later testwrap change that breaks this path fails before commit?
3. Do the non-ECPG regression suites pass through a required wrapper, given that the top-level meson.build change affects every test that uses testwrap?
4. Has the patch been tested on a Linux cross build with a qemu or wine exe_wrapper and no binfmt registration?
5. What is the third patch that message #2 and message #3 mention, and is it needed for the reported results?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the Meson build and the testwrap test harness would have to own it. Only the author has posted in the thread.
- **Liability:** The owner signs for the changed test() call in the top-level meson.build, which every test run uses, and for a wrapper path in testwrap that no regular CI run covers. Later testwrap changes can break that path without notice.
- **Author follow-up:** The author rebased to v2, posted further test results in message #3 and moved the entry to the next CommitFest, so he appears to stay engaged.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small and touches three build and test files, and it helps developers who cross-build with Meson. It has had no review in two CommitFests, so it needs a committer who knows the Meson build to look at the two open design points.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Minimum Meson version for the new calls is not stated. Refuted (no failure scenario): As far as I recall, Meson added meson.can_run_host_binaries() in 0.55, below the Postgres minimum of 0.57.2. Meson also warns at setup, on any version, when a project uses a feature newer than its declared meson_version, so a packager is not surprised.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] and [P7] add EXTRA_REGRESS_OPTS to testwrap, a different feature, and neither was turned down on design grounds. Message #1 cites a 2024 thread that suggested a cross file with needs_exe_wrapper=false, and the proposal builds on that suggestion, so no earlier rejection reason applies here.
- standards-compat: 2 objection(s). The change touches only the build and test system. No SQL, catalog, pg_dump, psql or wire-protocol surface is involved, so most of this angle does not apply.
- replication-ondisk: 0 objection(s). The proposal changes only the Meson build and the testwrap test harness. It touches no on-disk format, WAL record, replication path, catalog, wire protocol message, or extension-facing ABI, so this angle has nothing to object to.
- security: 0 objection(s). The proposal changes only how the Meson build runs ECPG and regression tests in cross builds. It adds no server privilege, role, data exposure or authentication path. MESON_EXE_WRAPPER comes from the developer's own cross file and environment, which already control the build, so the change gives no one new power.
- maintenance-fit: 1 objection(s). The change is small, it lives in the build system, and it uses a standard Meson check, so fit is not in doubt. The one maintenance concern above is fixable.
- already-exists: 0 objection(s). From the existing-feature angle there is nothing to raise. No Postgres option, Meson setting or external tool runs ECPG tests in a cross build today, because the ECPG meson.build skips them for every cross build. The patch reuses Meson's own exe_wrapper and can_run_host_binaries() and does not add a second mechanism. [P3] and [P7] also touch testwrap, but they are about EXTRA_REGRESS_OPTS, so neither applies.
- demand: 0 objection(s). Demand evidence is thin: only the author has posted, and the stated group is developers cross-building x64 on Windows ARM64. A demand objection still does not hold up, because the change only turns on existing test coverage. It adds no user-facing feature, SQL, GUC or catalog surface whose cost others carry. The shared testwrap change that every test run goes through belongs to the design angle.

### Precedents retrieved

- P1: [meson: Adjust test timeout for Valgrind builds](https://commitfest.postgresql.org/patch/6646/) (cf, Withdrawn, 2026-06-10) - judged not relevant; reason summarised from its thread
- P2: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant; reason summarised from its thread
- P3: [Add support for EXTRA_REGRESS_OPTS for meson test](https://commitfest.postgresql.org/patch/5602/) (cf, Returned with feedback, 2025-08-05) - used in the review; reason summarised from its thread
- P4: [Windows meson build](https://commitfest.postgresql.org/patch/5522/) (cf, Returned with feedback, 2026-03-27) - judged not relevant; reason summarised from its thread
- P5: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - judged not relevant; reason summarised from its thread
- P6: [\crosstabview (previously: \rotate) in psql for crosstab-style display](https://commitfest.postgresql.org/patch/372/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P7: [meson/testwrap: support EXTRA_REGRESS_OPTS environment variable](https://commitfest.postgresql.org/patch/6711/) (cf, Withdrawn, 2026-05-04) - used in the review; reason summarised from its thread
- P8: [Speed up clean meson builds by ~25%](https://commitfest.postgresql.org/patch/4914/) (cf, Rejected, 2024-05-20) - judged not relevant; reason summarised from its thread
- P9: [Improvements to Meson docs](https://commitfest.postgresql.org/patch/4208/) (cf, Returned with feedback, 2024-01-20) - judged not relevant
- P10: [Support UTF16/32 host variable in ECPG](https://commitfest.postgresql.org/patch/1337/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P11: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P12: [Revert pgindent changes to ecpg include files that are part of ecpg regession test output, and update pgindent script to avoid them in the future.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cc6ff45f8813593e7c8c0757cd13a7b63a41768) (revert, Reverted, 2010-02-26) - judged not relevant
- P13: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3560/) (cf, Returned with feedback, 2022-02-26) - judged not relevant
- P14: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/3561/) (cf, Returned with feedback, 2022-12-01) - judged not relevant
- P15: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P16: [Document hostssl related concepts better](https://commitfest.postgresql.org/patch/5549/) (cf, Withdrawn, 2025-03-09) - judged not relevant
- P17: [Add native windows on arm64 support](https://commitfest.postgresql.org/patch/4309/) (cf, Returned with feedback, 2023-09-19) - judged not relevant
- P18: [Atomic commit support for foreign data wrappers](https://commitfest.postgresql.org/patch/180/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P19: [Fix MINGW build on ARM64 on Windows](https://commitfest.postgresql.org/patch/5610/) (cf, Returned with feedback, 2025-04-07) - judged not relevant
- P20: [Validate user-supplied c_args in meson builds](https://commitfest.postgresql.org/patch/6596/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P21: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P22: [Revert error-throwing wrappers for the printf family of functions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c071936e94c6859afb2ec8d2c8dddf7bcdab7ee) (revert, Reverted, 2015-05-19) - judged not relevant
- P23: [Set host(no)ssl settings from initdb](https://commitfest.postgresql.org/patch/2378/) (cf, Returned with feedback, 2020-09-07) - judged not relevant
- P24: [Doc: Improve hostssl related descriptions and option presentation](https://commitfest.postgresql.org/patch/5088/) (cf, Withdrawn, 2025-02-27) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 107 s; cost reported by the CLI: $1.09
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 59 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:22:46.350Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
