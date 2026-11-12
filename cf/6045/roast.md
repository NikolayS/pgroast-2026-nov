# Design pre-review: Python/pytest test framework

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6045](https://commitfest.postgresql.org/patch/6045/); [thread](https://www.postgresql.org/message-id/flat/CAOYmi%2BkThkM9Z87u%3DR_Wi7fCor2i%2BUZKAyq0UCyprzCwTQvqgA%40mail.gmail.com), 116 of 116 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The thread shows demand and the goal has support, so level 1b holds. The design leaves three things open that the list must settle before code review. It has no buildfarm plan for installing dependencies and collecting results (prior-attempts-1), no rule that keeps ported tests running where pytest is absent (standards-compat-2), and no end state for the two-framework period (maintenance-fit-1). [P4] shows a Python change reverted because the buildfarm was not ready.

Not checked: Only the v1 and v2 patch prose was read, and the thread is at v13, so the current dependency handling and the Autoconf result path could not be checked. The quality of the fixtures, the libpq binding and the pgtap plugin was not judged.

## Prior attempts

- [Revert "Make configure prefer python3 to plain python."](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com) (related; Reverted, 2022-01-20). "The buildfarm is not ready for python3, evidently, so we'll give the owners some more time to get set up." Different now: P4 made configure prefer python3, and buildfarm owners were not ready. This proposal adds Python and third-party packages as a test dependency, with default auto.
- [Rewriting the test of pg_upgrade as a TAP test](https://www.postgresql.org/message-id/flat/CAB7nPqRdaN1A1YNjxNL9T1jUEWct8ttqq29dNv8W_o37+e8wfA@mail.gmail.com) (related; Returned with feedback, 2017-09-29). "I have reverted this because of the build farm issue. Putting the patch on hold in the CF until we have a new plan." ([message](https://www.postgresql.org/message-id/68eee9f1-864d-a6f8-cbf9-dd9bda6be9be%402ndquadrant.com)) Different now: P1 replaced one test script with a TAP test and was reverted over a build farm issue. This proposal adds a whole test framework, so the buildfarm readiness question is larger.
- [Rewriting the test of pg_upgrade as a TAP test - take two](https://www.postgresql.org/message-id/flat/20180126080026.GI17847@paquier.xyz) (related; Returned with feedback, 2018-04-07). "Basically, it imposes that from now on all program invocations must observe the bindir setting, which someone is surely going to forget." ([message](https://www.postgresql.org/message-id/c4c618c2-8ab4-df89-0f08-4f56cb4ac27c%402ndquadrant.com)) Different now: P3 needed a buildfarm client release before a new TAP suite could land. This proposal adds a new suite type, and the thread names no client change.

## Objections a reviewer is likely to raise

### 1. No stated plan for buildfarm client support of pytest suites

After commit, buildfarm animals where -Dpytest=auto finds pytest start running the new suites, while animals without it skip them without notice. Under Autoconf, pytest runs directly with colored output and no TAP or log file, so failures are not captured in a form the buildfarm client collects. Committers find out when failures appear that nobody can read.

Earlier test infrastructure changes were reverted or returned when the buildfarm was not ready: a new TAP suite for pg_upgrade [P1][P3], and configure's preference for python3 [P4]. Message #77 says the buildfarm client depends on test output being saved to files. Message #71 notes that a venv-only setup blocks packagers, and message #92 has the Autoconf path running pytest directly. The thread does not yet say how buildfarm owners install pytest and cryptography, or how the client reports pytest results.

Precedent: [Rewriting the test of pg_upgrade as a TAP test](https://www.postgresql.org/message-id/flat/CAB7nPqRdaN1A1YNjxNL9T1jUEWct8ttqq29dNv8W_o37+e8wfA@mail.gmail.com) (Returned with feedback, 2017-09-29)

Precedent: [Rewriting the test of pg_upgrade as a TAP test - take two](https://www.postgresql.org/message-id/flat/20180126080026.GI17847@paquier.xyz) (Returned with feedback, 2018-04-07)

Precedent: [Revert "Make configure prefer python3 to plain python."](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com) (Reverted, 2022-01-20)

**Question:** How will buildfarm animals install the dependencies and collect pytest results under both Meson and Autoconf, and does that need a client release before commit?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#71](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#77](https://www.postgresql.org/message-id/a38765d0-da19-4121-aaf9-cbf111da5461%40dunslane.net), [#92](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Ported Perl tests disappear for builds without pytest

A packager builds with tap_tests enabled but has no pytest, which Andres expects in message #71, so -Dpytest=auto turns itself off. When the load balance TAP test is replaced by a Python version, that coverage is gone from the package build with no warning. A libpq load-balancing regression then passes their tests.

The brief and v9-0003 convert the load_balance_hosts Perl test to Python. Message #92 says the plan now includes porting Perl tests at some point. Because the option defaults to auto, a ported test becomes a test that silently does not run wherever pytest is missing. Back branches keep the Perl version, so the same fix needs a different test on each branch.

**Question:** Until pytest is required, what stops a ported test from silently losing coverage on builds where pytest is absent, and will the Perl version stay alongside it?

<sub>fixable, level 2, design; messages [#71](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#92](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com), [#108](https://www.postgresql.org/message-id/froy7mvxq76o2asuyzajyvaqelzuv372xsylfvhlhkvis7ojr4%4023lg5adaxvdw), [#110](https://www.postgresql.org/message-id/DFJ7MRI3WIVU.MZQI2USE0SY7%40jeltef.nl); `standards-compat-2`</sub>

### 3. Pinned pytest version range ties configure to upstream release dates

A distribution moves to a pytest release outside the declared range. With -Dpytest=auto, the pytest suites quietly stop running on that platform. With -Dpytest=enabled, configure fails on a stable branch that nobody has touched, and someone must backpatch a bump to the version range.

The v2 requirements file pins pytest to at least 7.0 and below 9, and it adds cryptography with a lower bound that follows Debian releases. The brief notes that the project has not said which minimum Python and pytest versions it supports. Message #42 asks that new dependencies be judged on a 10- to 20-year view, but the patch has no policy for when a range is widened, or on which branches.

Precedent: [Revert "Make configure prefer python3 to plain python."](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com) (Reverted, 2022-01-20)

**Question:** Which Python, pytest and cryptography versions will each supported branch accept, and who widens the range on old branches when distributions move ahead?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#42](https://www.postgresql.org/message-id/1799088.1718291987%40sss.pgh.pa.us), [#71](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#92](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com); `standards-compat-3`</sub>

### 4. Two test frameworks with no stated migration plan

A committer changes server setup defaults or adds a helper in PostgreSQL::Test::Cluster. The pypg fixtures keep the old behaviour because no build step or test compares the two. Months later, the Perl and pytest suites disagree on the same feature, and a reviewer has to read both languages to find out why.

Tom Lane (message #42) and Robert Haas (messages #48, #68) objected to running two parallel frameworks with similar capabilities for ten to twenty years. Message #92 says side-by-side forever was outvoted, but no version up to v13 gives a porting plan, a timeline, or an end state. Without that, the patch commits the project to keeping two cluster-management layers in sync indefinitely. Message #55 notes that reverting test infrastructure once tests depend on it has rarely happened.

**Question:** Which Perl suites will be ported, in what order and by when, and what rule decides whether a new test goes in Perl or Python until then?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#42](https://www.postgresql.org/message-id/1799088.1718291987%40sss.pgh.pa.us), [#48](https://www.postgresql.org/message-id/CA%2BTgmoYvLLAJthL9ev-qc3gVLYX5WC3y3T5hsObaEnVOAx%2Be6g%40mail.gmail.com), [#55](https://www.postgresql.org/message-id/CA%2BTgmoYZ-HhwGKqrMaPtnuwq8jWMf_XJVOnDB%3D8aD%2BY73SeXEA%40mail.gmail.com), [#68](https://www.postgresql.org/message-id/CA%2BTgmoZV8B35Mvx8DqHtJjHDQxscJhwOWzNWiPj%2BdsUhwaUfSg%40mail.gmail.com), [#92](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. How will buildfarm animals install pytest and cryptography and collect pytest results under both Meson and Autoconf, and does that need a buildfarm client release first?
2. Until pytest is required, will the Perl version of a ported test stay, so that builds without pytest keep that coverage?
3. Which Perl suites will be ported, in what order, and what rule decides whether a new test is written in Perl or Python until then?
4. Which minimum Python, pytest and cryptography versions will each supported branch accept, and who widens the range on old branches?
5. How do packagers get pytest and cryptography without a virtualenv, the concern Andres Freund raised?

## Touch points

- **ABI / extension API:** The shared Python package wraps libpq in a hand-written ctypes binding that depends on the libpq ABI.
- **security / privileges:** pytest and cryptography are third-party packages on developer and buildfarm machines; message #1 leaves supply-chain safety open.
- **backpatchability:** A test ported to Python differs from the Perl version on back branches, and version ranges need care on old branches.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a second test framework, a pgtap plugin, cluster fixtures, and --enable-pytest and -Dpytest build options.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund takes part in the visible thread and supports moving off Perl, but no committer has said they will commit it. The owner must know the build system, the test infrastructure and the buildfarm client.
- Priority: fix design first; audience a sizeable minority; review cost high. Those who gain are developers who write tests, and several asked for this independently in the thread. The patch touches configure, Meson, CI and a new Python package, and the buildfarm and migration plans are still missing after 116 messages.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 7.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
