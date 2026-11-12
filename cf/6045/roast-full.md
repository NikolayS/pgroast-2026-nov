# Design pre-review: Python/pytest test framework

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6045](https://commitfest.postgresql.org/patch/6045/) (Needs review); authors: Jacob Champion (jchampion), Jelte Fennema-Nio (jeltef)
- Thread: [RFC: adding pytest as a supported test framework](https://www.postgresql.org/message-id/flat/CAOYmi%2BkThkM9Z87u%3DR_Wi7fCor2i%2BUZKAyq0UCyprzCwTQvqgA%40mail.gmail.com), started 2024-06-10
- Messages read: 116 of 116
- Patch prose read: v2-0001-meson-Include-TAP-tests-in-the-configuration-summ.patch, v2-0002-Add-support-for-pytest-test-suites.patch, v2-0003-WIP-pytest-Add-some-SSL-client-tests.patch, v2-0004-WIP-pytest-Add-some-server-side-SSL-tests.patch, v2-0005-ci-Add-MTEST_SUITES-for-optional-test-tailoring.patch, v2-0006-XXX-run-pytest-and-ssl-suite-all-OSes.patch, v1-0001-meson-Include-TAP-tests-in-the-configuration-summ.patch, v1-0002-Add-support-for-pytest-test-suites.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The thread shows demand and the goal has support, so level 1b holds. The design leaves three things open that the list must settle before code review. It has no buildfarm plan for installing dependencies and collecting results (prior-attempts-1), no rule that keeps ported tests running where pytest is absent (standards-compat-2), and no end state for the two-framework period (maintenance-fit-1). [P4] shows a Python change reverted because the buildfarm was not ready.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only the v1 and v2 patch prose was read, and the thread is at v13, so the current dependency handling and the Autoconf result path could not be checked. The quality of the fixtures, the libpq binding and the pgtap plugin was not judged.

## The proposal, restated

**Problem.** Developers cannot rerun one failing TAP test alone, and failure output rarely shows what failed without reading logs. Few newer contributors know Perl, so they avoid writing tests, and the Perl suite gives little room for modern tooling or for testing client and server separately, for example against a mock peer.

**Proposed design.** Configure and Meson gain an option that runs Python test suites under pytest, with a custom plugin that turns pytest results into TAP output for the Meson test runner. A shared Python package adds fixtures that start and stop clusters, run queries through a ctypes binding to libpq, and map server errors to Python exceptions. Later patches port the load balance tests and parts of the SSL tests.

**User-visible surface.**
- Configure option --enable-pytest and Meson option -Dpytest (feature, default auto).
- Meson option PYTEST, the path to the pytest binary.
- Meson option tap_tests description changed to say the TAP tests are Perl-based.
- New test tree src/test/pytest with a shared Python fixtures package and a pgtap output plugin.
- Python package dependencies: pytest, plus cryptography for the SSL tests.

**What the author claims.**
- pytest fixtures let each test declare its own setup, so a single failing test can be rerun in isolation.
- Plain assert statements show the compared values on failure, with no comparison DSL to learn.
- Server logs are attached to the failure report, split by setup, test and teardown.
- The Perl tests stay in place, and the pytest entry point can be reverted cleanly if it does not work out.
- In-person meetings showed support or neutrality and no strong objections.

**Thread state.** Andres Freund wants central infrastructure first and a smaller first patch; Jelte cut the patch set down in v9 and has rebased it through v13. Jacob plans no fast-track and wants buy-in from several maintainers, and no committer has said they will commit it.

## 2. Prior attempts

### Revert "Make configure prefer python3 to plain python." (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e)
- **Thread:** [https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com)
- **Date:** decided 2022-01-20; started 2022-01-19
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit f201da39edcd6ac1ab9a3edf3e20e2a73bbbe69e. The buildfarm is not ready for python3, evidently, so we'll give the owners some more time to get set up.
- **Reason (quote):** "The buildfarm is not ready for python3, evidently, so we'll give the owners some more time to get set up."
- **What is different this time:** P4 made configure prefer python3, and buildfarm owners were not ready. This proposal adds Python and third-party packages as a test dependency, with default auto.

### Rewriting the test of pg_upgrade as a TAP test (related)

- **Link:** [https://commitfest.postgresql.org/patch/1101/](https://commitfest.postgresql.org/patch/1101/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAB7nPqRdaN1A1YNjxNL9T1jUEWct8ttqq29dNv8W_o37+e8wfA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAB7nPqRdaN1A1YNjxNL9T1jUEWct8ttqq29dNv8W_o37+e8wfA@mail.gmail.com)
- **Date:** decided 2017-09-29; started 2017-04-03
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut committed the basic TAP test, then reverted it because of a build farm issue and set the entry to Returned with feedback until there was a new plan. Two points were also still open: the patch removed the documented cross-version upgrade testing, and there was no plan for build farm animals that call vcregress.pl upgradecheck.
- **Reason (quote):** "I have reverted this because of the build farm issue. Putting the patch on hold in the CF until we have a new plan." ([message](https://www.postgresql.org/message-id/68eee9f1-864d-a6f8-cbf9-dd9bda6be9be%402ndquadrant.com))
- **What is different this time:** P1 replaced one test script with a TAP test and was reverted over a build farm issue. This proposal adds a whole test framework, so the buildfarm readiness question is larger.

### Rewriting the test of pg_upgrade as a TAP test - take two (related)

- **Link:** [https://commitfest.postgresql.org/patch/1497/](https://commitfest.postgresql.org/patch/1497/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20180126080026.GI17847@paquier.xyz](https://www.postgresql.org/message-id/flat/20180126080026.GI17847@paquier.xyz)
- **Date:** decided 2018-04-07; started 2018-01-26
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** A new TAP suite in the pg_upgrade directory would make the buildfarm client run the pg_upgrade tests twice, so a buildfarm client release was needed first, and the author judged the commitfest window too short for v11. Peter Eisentraut also found the PostgresNode.pm bindir refactoring too invasive, because every program call must now observe the bindir setting and future code can forget it. Cross-version runs still failed to load regress.so and related libraries after the upgrade.
- **Reason (quote):** "Basically, it imposes that from now on all program invocations must observe the bindir setting, which someone is surely going to forget." ([message](https://www.postgresql.org/message-id/c4c618c2-8ab4-df89-0f08-4f56cb4ac27c%402ndquadrant.com))
- **What is different this time:** P3 needed a buildfarm client release before a new TAP suite could land. This proposal adds a new suite type, and the thread names no client change.


## 3. Objections a reviewer is likely to raise

### 1. No stated plan for buildfarm client support of pytest suites

fixable · level 2, rubric item 13 · maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

After commit, buildfarm animals where -Dpytest=auto finds pytest start running the new suites, while animals without it skip them without notice. Under Autoconf, pytest runs directly with colored output and no TAP or log file, so failures are not captured in a form the buildfarm client collects. Committers find out when failures appear that nobody can read.

Earlier test infrastructure changes were reverted or returned when the buildfarm was not ready: a new TAP suite for pg_upgrade [P1][P3], and configure's preference for python3 [P4]. Message #77 says the buildfarm client depends on test output being saved to files. Message #71 notes that a venv-only setup blocks packagers, and message #92 has the Autoconf path running pytest directly. The thread does not yet say how buildfarm owners install pytest and cryptography, or how the client reports pytest results.

**Precedents.**
- [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (Returned with feedback, 2017-09-29); thread: [link](https://www.postgresql.org/message-id/flat/CAB7nPqRdaN1A1YNjxNL9T1jUEWct8ttqq29dNv8W_o37+e8wfA@mail.gmail.com)
- [Rewriting the test of pg_upgrade as a TAP test - take two](https://commitfest.postgresql.org/patch/1497/) (Returned with feedback, 2018-04-07); thread: [link](https://www.postgresql.org/message-id/flat/20180126080026.GI17847@paquier.xyz)
- [Revert "Make configure prefer python3 to plain python."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e) (Reverted, 2022-01-20); thread: [link](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com)

Thread: [#71 (Andres Freund, 2024-06-14)](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#77 (Andrew Dunstan, 2024-06-15)](https://www.postgresql.org/message-id/a38765d0-da19-4121-aaf9-cbf111da5461%40dunslane.net), [#92 (Jacob Champion, 2025-09-09)](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com)

**Question:** How will buildfarm animals install the dependencies and collect pytest results under both Meson and Autoconf, and does that need a client release before commit?

Counter-check: held. No message through v13 says how buildfarm animals install pytest and cryptography or how the client collects pytest results, and message #77 says the client relies on stored log files. [P4] applies directly; [P1] and [P3] support only the point about buildfarm client readiness.

### 2. Ported Perl tests disappear for builds without pytest

fixable · level 2, rubric item 13 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `standards-compat-2`

A packager builds with tap_tests enabled but has no pytest, which Andres expects in message #71, so -Dpytest=auto turns itself off. When the load balance TAP test is replaced by a Python version, that coverage is gone from the package build with no warning. A libpq load-balancing regression then passes their tests.

The brief and v9-0003 convert the load_balance_hosts Perl test to Python. Message #92 says the plan now includes porting Perl tests at some point. Because the option defaults to auto, a ported test becomes a test that silently does not run wherever pytest is missing. Back branches keep the Perl version, so the same fix needs a different test on each branch.

Thread: [#71 (Andres Freund, 2024-06-14)](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#92 (Jacob Champion, 2025-09-09)](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com), [#108 (Andres Freund, 2026-01-07)](https://www.postgresql.org/message-id/froy7mvxq76o2asuyzajyvaqelzuv372xsylfvhlhkvis7ojr4%4023lg5adaxvdw), [#110 ("Jelte Fennema-Nio", 2026-01-08)](https://www.postgresql.org/message-id/DFJ7MRI3WIVU.MZQI2USE0SY7%40jeltef.nl)

**Question:** Until pytest is required, what stops a ported test from silently losing coverage on builds where pytest is absent, and will the Perl version stay alongside it?

Counter-check: held. The `auto` default copies `tap_tests`, and v9 labels the load balance port a POC after message #108 asked not to remove an existing test. But message #92 expects ports later, and no message says the Perl version stays until pytest is required.

### 3. Pinned pytest version range ties configure to upstream release dates

fixable · level 2, rubric item 15 · maintenance burden ("who maintains this in 10 years"), backpatchability · `standards-compat-3`

A distribution moves to a pytest release outside the declared range. With -Dpytest=auto, the pytest suites quietly stop running on that platform. With -Dpytest=enabled, configure fails on a stable branch that nobody has touched, and someone must backpatch a bump to the version range.

The v2 requirements file pins pytest to at least 7.0 and below 9, and it adds cryptography with a lower bound that follows Debian releases. The brief notes that the project has not said which minimum Python and pytest versions it supports. Message #42 asks that new dependencies be judged on a 10- to 20-year view, but the patch has no policy for when a range is widened, or on which branches.

**Precedents.**
- [Revert "Make configure prefer python3 to plain python."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e) (Reverted, 2022-01-20); thread: [link](https://www.postgresql.org/message-id/flat/2872c9a0-4b0a-1354-d5f6-94d6f85ba354@enterprisedb.com)

Thread: [#42 (Tom Lane, 2024-06-13)](https://www.postgresql.org/message-id/1799088.1718291987%40sss.pgh.pa.us), [#71 (Andres Freund, 2024-06-14)](https://www.postgresql.org/message-id/20240614161111.7o5gbmwezjithwjn%40awork3.anarazel.de), [#92 (Jacob Champion, 2025-09-09)](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com)

**Question:** Which Python, pytest and cryptography versions will each supported branch accept, and who widens the range on old branches when distributions move ahead?

Counter-check: held. Message #101 replaced the v2 requirements file and check script with pyproject.toml and importorskip, so the configure failure may no longer happen as described. The thread still names no supported Python, pytest and cryptography versions per branch, so the version policy question stands.

### 4. Two test frameworks with no stated migration plan

fixable · level 1b, rubric item 5 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer changes server setup defaults or adds a helper in PostgreSQL::Test::Cluster. The pypg fixtures keep the old behaviour because no build step or test compares the two. Months later, the Perl and pytest suites disagree on the same feature, and a reviewer has to read both languages to find out why.

Tom Lane (message #42) and Robert Haas (messages #48, #68) objected to running two parallel frameworks with similar capabilities for ten to twenty years. Message #92 says side-by-side forever was outvoted, but no version up to v13 gives a porting plan, a timeline, or an end state. Without that, the patch commits the project to keeping two cluster-management layers in sync indefinitely. Message #55 notes that reverting test infrastructure once tests depend on it has rarely happened.

Thread: [#42 (Tom Lane, 2024-06-13)](https://www.postgresql.org/message-id/1799088.1718291987%40sss.pgh.pa.us), [#48 (Robert Haas, 2024-06-13)](https://www.postgresql.org/message-id/CA%2BTgmoYvLLAJthL9ev-qc3gVLYX5WC3y3T5hsObaEnVOAx%2Be6g%40mail.gmail.com), [#55 (Robert Haas, 2024-06-13)](https://www.postgresql.org/message-id/CA%2BTgmoYZ-HhwGKqrMaPtnuwq8jWMf_XJVOnDB%3D8aD%2BY73SeXEA%40mail.gmail.com), [#68 (Robert Haas, 2024-06-14)](https://www.postgresql.org/message-id/CA%2BTgmoZV8B35Mvx8DqHtJjHDQxscJhwOWzNWiPj%2BdsUhwaUfSg%40mail.gmail.com), [#92 (Jacob Champion, 2025-09-09)](https://www.postgresql.org/message-id/CAOYmi%2BnEqA2LmetcJKUDmctypPLLumkVwj3vQ3idYd8yAGza5Q%40mail.gmail.com)

**Question:** Which Perl suites will be ported, in what order and by when, and what rule decides whether a new test goes in Perl or Python until then?

Counter-check: held. Robert Haas accepted the two-framework cost in principle in messages #55 and #68, which lowers the weight of this objection. But message #92 says ports are now expected, and no later version gives an order, an end state, or a rule for new tests.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The shared Python package wraps libpq in a hand-written ctypes binding that depends on the libpq ABI. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | pytest and cryptography are third-party packages on developer and buildfarm machines; message #1 leaves supply-chain safety open. |
| backpatchability | **yes** | A test ported to Python differs from the Perl version on back branches, and version ranges need care on old branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a second test framework, a pgtap plugin, cluster fixtures, and --enable-pytest and -Dpytest build options. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How will buildfarm animals install pytest and cryptography and collect pytest results under both Meson and Autoconf, and does that need a buildfarm client release first?
2. Until pytest is required, will the Perl version of a ported test stay, so that builds without pytest keep that coverage?
3. Which Perl suites will be ported, in what order, and what rule decides whether a new test is written in Perl or Python until then?
4. Which minimum Python, pytest and cryptography versions will each supported branch accept, and who widens the range on old branches?
5. How do packagers get pytest and cryptography without a virtualenv, the concern Andres Freund raised?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund takes part in the visible thread and supports moving off Perl, but no committer has said they will commit it. The owner must know the build system, the test infrastructure and the buildfarm client.
- **Liability:** The owner signs for a second test framework on every supported branch for years: the build options, the pgtap plugin, the cluster fixtures, the libpq binding, and buildfarm failures caused by Python package versions.
- **Author follow-up:** Good sign: Jelte Fennema-Nio cut the patch set down in v9 and rebased it through v13, and both authors have kept the entry active over six CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Those who gain are developers who write tests, and several asked for this independently in the thread. The patch touches configure, Meson, CI and a new Python package, and the buildfarm and migration plans are still missing after 116 messages.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Autoconf path runs pytest outside the TAP result format. Refuted (not specific): Duplicate of prior-attempts-1, which covers the Autoconf result collection and also the dependency installation.
- `maintenance-fit-2` Default auto option can silently drop ported coverage on buildfarm. Refuted (not specific): Duplicate of standards-compat-2 for the silent loss of ported coverage, and of prior-attempts-1 for the buildfarm dependencies.
- `maintenance-fit-3` Hand-written ctypes libpq binding needs manual syncing. Refuted (no failure scenario): A new libpq function that has no binding yet breaks nothing, because the test that first needs it adds the binding, and existing libpq signatures stay stable. Message #101 says the parsing problems from message #99 were addressed, and Robert Haas asked for a libpq FFI in message #52.
- `already-exists-1` Most technical gains look reachable in the existing Perl TAP framework. Refuted (already handled): Messages #3, #38, #43 and #87 show the need is common, and Perl improvements cannot fix contributors avoiding Perl. Robert Haas weighed this in messages #48 and #59 and found improving Perl alone unrealistic; [P8] rejected a Perl protocol client and does not apply.
- `demand-1` Costs fall on all committers and buildfarm owners, with no plan to end them. Refuted (not specific): Duplicate of maintenance-fit-1 for the missing porting plan, and of standards-compat-2 for the skipped suites. Its claim that the two-framework cost has no recorded answer is also inaccurate, because messages #54, #55, #68 and #90 discuss it.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes the same idea. The listed rows share only the buildfarm-readiness reason, which is raised above.
- standards-compat: 3 objection(s). There is no SQL, catalog or psql surface, so the standards angle covers only build options, dependencies and buildfarm integration.
- replication-ondisk: 0 objection(s). The proposal adds test infrastructure only: build options, a pytest runner, Python fixtures and a ctypes libpq binding. It changes no on-disk format, WAL record, catalog, wire protocol message or extension ABI, so this angle has nothing to object to.
- security: 0 objection(s). No objection from the security angle. The proposal adds a developer and CI test framework that is not installed or shipped, so it gives no database role new abilities and exposes nothing to users of a running server. Its trust-auth clusters and test listeners follow what the Perl TAP suite already does. The third-party packages (pytest, cryptography) run only on developer, packager and buildfarm machines. Message #1 already names supply-chain review as an open topic, and v9 dropped the automatic .venv detection. What could not be checked without the code: whether mock servers bind only to localhost, and whether temporary socket directories are private to the test user.
- maintenance-fit: 3 objection(s). Objections raised; no committer has yet said they will own the result.
- already-exists: 1 objection(s). One objection only. External tools such as testgres serve out-of-tree use and do not replace in-tree infrastructure, so they give no further objection.
- demand: 1 objection(s). Demand from developers is shown by several independent people in the thread, two in-person meetings, and existing pytest suites outside core. The one objection concerns who pays for that demand.

### Precedents retrieved

- P1: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - used in the review; reason summarised from its thread
- P2: [Rewriting the test of pg_upgrade as a TAP test - take three](https://commitfest.postgresql.org/patch/3120/) (cf, Withdrawn, 2021-11-18) - judged not relevant; reason summarised from its thread
- P3: [Rewriting the test of pg_upgrade as a TAP test - take two](https://commitfest.postgresql.org/patch/1497/) (cf, Returned with feedback, 2018-04-07) - used in the review; reason summarised from its thread
- P4: [Revert "Make configure prefer python3 to plain python."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=512fc2dd790e3e7c35b3af8cd7586aeb25248c6e) (revert, Reverted, 2022-01-20) - used in the review
- P5: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P6: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P7: [Support migration of plpythonu to Python 3](https://commitfest.postgresql.org/patch/2473/) (cf, Withdrawn, 2020-04-07) - judged not relevant; reason summarised from its thread
- P8: [TAP test module - PostgresClient](https://commitfest.postgresql.org/patch/1442/) (cf, Rejected, 2018-03-03) - used in the review; reason summarised from its thread
- P9: [Backport perl tests for pg_upgrade](https://commitfest.postgresql.org/patch/3997/) (cf, Withdrawn, 2022-12-19) - judged not relevant; reason summarised from its thread
- P10: [Add ssl_protocols configuration option](https://commitfest.postgresql.org/patch/11/) (cf, Rejected, 2015-02-13) - judged not relevant; reason summarised from its thread
- P11: [allow building trusted languages without the untrusted versions](https://commitfest.postgresql.org/patch/3647/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P12: [Revert all new recent changes to add PGXS options for TAP and isolation](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d7dd18686d0373289a3bf2b3669edae20875cbc) (revert, Reverted, 2018-11-26) - used in the review
- P13: [Regression tests for V2 protocol.](https://commitfest.postgresql.org/patch/1303/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P14: [possibility to write window function in PL languages (like PLpgSQL)](https://commitfest.postgresql.org/patch/2697/) (cf, Withdrawn, 2021-01-27) - judged not relevant
- P15: [Detect test suite sideeffects in pg_regress](https://commitfest.postgresql.org/patch/4321/) (cf, Withdrawn, 2024-04-03) - judged not relevant
- P16: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P17: [Revert "In the pg_upgrade test suite, don't write to src/test/regress."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ae35e1c9d74afc3b1e7572ad60781946fdf71e95) (revert, Reverted, 2019-05-19) - judged not relevant
- P18: [Revert "Add GUC checks for ssl_min_protocol_version and ssl_max_protocol_version"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=414c2fd1e1c0ccd1af1cad046bef1264e3f4347f) (revert, Reverted, 2020-02-06) - judged not relevant
- P19: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P20: [PROXY protocol support](https://commitfest.postgresql.org/patch/3032/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Revert 1fbc30f1dca6624c01a9a6a9ff711a7036c68532 because the --enable-shared is about Plyton's configure, not ours.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39d0653d058b87b437be05fd295ceb5bb4e9e05c) (revert, Reverted, 2012-08-30) - judged not relevant
- P22: [Revert "Replace use of deprecated Python module distutils.sysconfig"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dda42ff8e60541e9eb5ac21b79af32d914f408a8) (revert, Reverted, 2022-01-18) - judged not relevant
- P23: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P24: [Revert to using --enable-auto-import in Cygwin builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8d6e2d4abf77c422714448e5f4270fdb1a84d973) (revert, Reverted, 2014-02-16) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/CAOYmi%2BkThkM9Z87u%3DR_Wi7fCor2i%2BUZKAyq0UCyprzCwTQvqgA%40mail.gmail.com). Others: https://www.postgresql.org/message-id/flat/DKSSP47Y857Z.1FUU91WMDZWPZ%40jeltef.nl
- attachment limit (8) reached; v1-0003-WIP-pytest-Add-some-SSL-client-tests.patch not read
- attachment limit (8) reached; v1-0004-WIP-pytest-Add-some-server-side-SSL-tests.patch not read
- attachment limit (8) reached; v1-0005-ci-Add-MTEST_SUITES-for-optional-test-tailoring.patch not read
- attachment limit (8) reached; v1-0006-XXX-run-pytest-and-ssl-suite-all-OSes.patch not read
- 27 long messages in the middle of the thread were left out of the prompts
- review angle "already-exists" returned prior attempts; ignored (only the prior-attempts angle may)

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 185 s; cost reported by the CLI: $7.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 98 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:40:07.557Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
