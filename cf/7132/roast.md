# Design pre-review: meson: Support ECPG tests in runnable cross builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7132](https://commitfest.postgresql.org/patch/7132/); [thread](https://www.postgresql.org/message-id/flat/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No objection survived at level 1a or 1b; the goal is clear and the change only turns on existing test coverage. Two fixable level 2 points remain. standards-compat-1 asks whether programs started by pg_regress_ecpg run when a wrapper is required, and maintenance-fit-1 asks what keeps the wrapper path in testwrap tested. A committer who knows the Meson build has to judge both, and none has posted.

Not checked: Without the code, we could not check how testwrap resolves the runner path and builds the wrapper command, or whether the changed test() call alters native builds. Whether pg_regress_ecpg passes the wrapper to the programs it starts also needs the code.

## Prior attempts

None found before 2026-07-24.

## Objections a reviewer is likely to raise

### 1. Wrapper covers the runner but not the programs it starts

A developer cross-builds for aarch64 on x86-64 Linux with exe_wrapper = qemu-aarch64 and no binfmt registration. After the patch the ECPG tests run, and pg_regress_ecpg starts under qemu but likely spawns initdb, postgres and the ECPG test programs without the wrapper. Every test fails, and the build turns red with no configuration change.

Per the commit message, testwrap wraps only the runner, but meson.can_run_host_binaries() also returns true when a wrapper is required. Every wrapper case tested (messages #1 and #3) ran on Windows, where x64 runs natively or through emulation and the Python wrapper only passes calls through. That shows the runner goes through the wrapper, but it does not show that its child processes can run. Linux cross setups with qemu or wine are untested.

**Question:** In a cross build where host binaries run only through the wrapper, how do initdb, postgres and the ECPG test programs started by pg_regress_ecpg get run?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAAH-eSWwQr0P1p-JvKnEuco0-biwcxfYYx%3D3yUK5UEqCWc9%3D9w%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAAH-eSUT%3DQaEtzTcDFk7iKEhBsXm-i_6WgfTCjXKF%2BgmmWFH6A%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Wrapper path in testwrap has no recurring test coverage

A later patch changes testwrap's argument or directory handling, for example the pending EXTRA_REGRESS_OPTS work. Native CI passes because MESON_EXE_WRAPPER is unset there, but the wrapper branch of testwrap is broken. The next person to cross-build with an exe_wrapper finds it months later, with no committer who knows why the branch exists.

Patch 1 adds a second invocation path to src/tools/testwrap, and this path runs only when Meson sets MESON_EXE_WRAPPER. All results in messages #1 to #3 come from the author's own Windows runs, and nothing proposes a CI task or buildfarm animal that sets this path up. Other testwrap changes are in flight, so later edits are likely to reach this branch without anyone running it. No committer has posted in the thread, so ownership of the branch is not settled.

**Question:** Which CI task or buildfarm animal would run regression tests through an exe_wrapper, so a later testwrap change that breaks this path fails before commit?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAAH-eSUHr2UgYvYV%2BWotOLORF9OUNGKkimJEwAPAitWx0cgAtA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAAH-eSWwQr0P1p-JvKnEuco0-biwcxfYYx%3D3yUK5UEqCWc9%3D9w%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAAH-eSUT%3DQaEtzTcDFk7iKEhBsXm-i_6WgfTCjXKF%2BgmmWFH6A%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. In a cross build where host binaries run only through the wrapper, how do initdb, postgres and the ECPG test programs started by pg_regress_ecpg get run?
2. Which CI task or buildfarm animal would run regression tests through an exe_wrapper, so a later testwrap change that breaks this path fails before commit?
3. Do the non-ECPG regression suites pass through a required wrapper, given that the top-level meson.build change affects every test that uses testwrap?
4. Has the patch been tested on a Linux cross build with a qemu or wine exe_wrapper and no binfmt registration?
5. What is the third patch that message #2 and message #3 mention, and is it needed for the reported results?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** Patch 1 adds a wrapper path to src/tools/testwrap that runs only when Meson sets MESON_EXE_WRAPPER.

## Who would commit it, and when

- Owner: silent. Someone who knows the Meson build and the testwrap test harness would have to own it. Only the author has posted in the thread.
- Priority: needs champion; audience niche; review cost low. The patch is small and touches three build and test files, and it helps developers who cross-build with Meson. It has had no review in two CommitFests, so it needs a committer who knows the Meson build to look at the two open design points.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
