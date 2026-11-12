# Design pre-review: Windows locale and test portability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7193](https://commitfest.postgresql.org/patch/7193/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Windows locales and tests portability](https://www.postgresql.org/message-id/flat/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru), started 2026-02-06
- Messages read: 5 of 5
- Patch prose read: 0001-Fix-TAP-tests-for-Windows-with-non-C-numeric-locales.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the goal of passing TAP tests on comma-locale Windows has two independent reporters. The open points are fixable design questions. maintenance-fit-2 asks whether public Cluster.pm helpers should exist while a frontend fix that makes them unnecessary is under discussion. maintenance-fit-1 notes that nothing fails on common platforms when a new test skips the helper. demand-1 is weaker, because the change is test-only and costs users nothing.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the caching of the detected separator, the behaviour on Unix with a non-C LC_NUMERIC, and the handling of other separators could not be checked. Messages #4 and #5 were not shown in full.

## The proposal, restated

**Problem.** TAP tests fail on Windows machines whose regional settings use a comma as decimal separator, such as Russian. MSVC strtod() takes its locale from Windows regional settings and ignores the LC_NUMERIC=C that PostgreSQL::Test::Utils sets, so psql rejects \watch 0.01 and pg_test_timing prints commas. This affects buildfarm animals and developers with such locales.

**Proposed design.** The patch adds get_decimal_separator() and format_locale_number() to PostgreSQL::Test::Cluster. They detect the decimal separator by asking psql with numericlocale enabled, cache it per node, and format numbers for \watch. pg_test_timing's test regex accepts both '.' and ','.

**User-visible surface.**
- New Perl test methods PostgreSQL::Test::Cluster::get_decimal_separator() and format_locale_number(), available to extension TAP tests.

**What the author claims.**
- The fix works on the author's Russian-locale Windows 11 box with Strawberry Perl 5.42 and on CI machines.
- Accepting [,.] covers most comma locales, including Russian, Brazilian, German, Italian and Polish.
- The author calls the fix "ugly" and offers changing the machine locale as an alternative.

**Thread state.** Thomas Munro suggested reading LC_* variables with getenv() and passing them to setlocale() on Windows; Andrew Bille posted such a frontend patch in a separate thread. Both agree the test fix can go in independently, but nobody has reviewed this patch's code.

## 2. Prior attempts

### tsearch and tsdicts regression tests fail in Turkish locale on glibc (related)

- **Link:** [https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search)
- **Date:** decision date unknown
- **Outcome:** Not worth doing (do_not_want)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P7 was about regression tests failing under a Turkish locale on glibc and records no reason. This patch covers TAP tests under comma-decimal locales on Windows.


## 3. Objections a reviewer is likely to raise

### 1. Test API may become dead weight if frontend locale handling changes

fixable · level 2, rubric item 6 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

The test helpers ship in a release and extension TAP suites start calling get_decimal_separator(). Later, Andrew Bille's frontend patch makes psql honor LC_NUMERIC=C on Windows, so the helpers always return '.'. Core cannot remove them without breaking the extension tests that call them.

Messages #2, #3 and #5 treat the cause as frontend locale initialization on Windows. Message #3 shows the same \watch failure for real users, and the thread still has an open fix for it. The patch adds public methods to PostgreSQL::Test::Cluster, which extension tests use, to work around behaviour that may change soon. Messages #4 and #5 agree the two fixes are independent, but they do not discuss whether the test-side API should be permanent once the frontend fix lands.

Thread: [#2 (Thomas Munro, 2026-02-06)](https://www.postgresql.org/message-id/CA%2BhUKGK8Mru903e%2BETdi8-cuoMLDFcggyE4ZzSuJtu9oVaUZmg%40mail.gmail.com), [#3 (Andrew Bille, 2026-09-22)](https://www.postgresql.org/message-id/CAJnzarxD%3DFPRA14BVyStYWgr9HekapRU%2B_uDOmdNg%2BQF5McmKg%40mail.gmail.com), [#4 (Andrey Borodin, 2026-09-22)](https://www.postgresql.org/message-id/A733EF40-6E1A-4670-BF83-5E729434340C%40yandex-team.ru), [#5 (Andrew Bille, 2026-09-22)](https://www.postgresql.org/message-id/CAJnzary%2BNkyfLF%2B5CQKSprAmTyftReURkBLqrcVqF-5zasrb3Q%40mail.gmail.com)

**Question:** If the frontend fix is committed, can these helpers be removed, and could the test fix be kept private to the affected tests so that extensions do not start depending on them?

Counter-check: held. The patch adds public methods to Cluster.pm, which extension tests can call, while a frontend fix that would make them unnecessary is under discussion. The thread does not say whether the helpers could stay private to the affected tests.

### 2. New tests with fractional \watch values will silently skip the helper

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A developer later adds a TAP test that runs `\watch 0.1` as a plain literal. It passes on Linux CI and on most Windows animals, then fails only on a comma-locale Windows animal, possibly weeks later. Someone without that setup has to work out the cause and retrofit format_locale_number().

The patch fixes five existing call sites by hand, so every future test that passes a fractional number to psql or parses a frontend's decimal output must remember the helper. On common platforms nothing fails when a test leaves it out. [P6] is a case where a test was removed because carrying platform-specific locale behaviour in tests 'in perpetuity' was judged not worth it. The thread does not say how the five files were found or how new cases would be caught.

**Precedents.**
- [Revert test case added by commit 1e165d05fe06a9072867607886f818bc255507db.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=32ca22b02da9d8088b58b3dc64ad78464c7513a3) (Reverted, 2017-08-02); thread: [link](https://www.postgresql.org/message-id/flat/CAKKotZS-wcDcofXDCH=sidiuajE+nqHn2CGjLLX78anyDmi3gQ@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-02-06)](https://www.postgresql.org/message-id/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru)

**Question:** What makes a new test that passes a plain fractional literal to psql fail on Linux CI, rather than only on a comma-locale Windows animal?

Counter-check: held. The patch fixes call sites by hand, and nothing on Unix or dot-locale Windows fails when a new test skips the helper. [P6] concerned multiple expected files for a CREATE COLLATION error message, so it should not be used as support.

### 3. Small affected group with a known workaround; show more demand first

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A test author later adds `\watch 0.5` without format_locale_number(). Every Unix and dot-locale animal stays green. The test breaks only on a comma-locale Windows box, which the patch exists to support. That box may be one machine, and its owner can switch Windows regional settings to a dot locale.

The demand shown is two developers with Russian-locale Windows machines (messages #1, #3) and one planned buildfarm animal. Message #1 offers changing the machine locale as an alternative and calls the fix "ugly". Andrew Bille's frontend LC_NUMERIC patch (messages #3, #5) would fix both the `\watch` failure and pg_test_timing output, which would make the new Cluster.pm helpers unnecessary. [P6] records the same judgement: carrying test machinery for Windows locale variation was not worth it once the code path worked.

**Precedents.**
- [Revert test case added by commit 1e165d05fe06a9072867607886f818bc255507db.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=32ca22b02da9d8088b58b3dc64ad78464c7513a3) (Reverted, 2017-08-02); thread: [link](https://www.postgresql.org/message-id/flat/CAKKotZS-wcDcofXDCH=sidiuajE+nqHn2CGjLLX78anyDmi3gQ@mail.gmail.com)

Thread: [#1 (Andrey Borodin, 2026-02-06)](https://www.postgresql.org/message-id/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru), [#3 (Andrew Bille, 2026-09-22)](https://www.postgresql.org/message-id/CAJnzarxD%3DFPRA14BVyStYWgr9HekapRU%2B_uDOmdNg%2BQF5McmKg%40mail.gmail.com), [#5 (Andrew Bille, 2026-09-22)](https://www.postgresql.org/message-id/CAJnzary%2BNkyfLF%2B5CQKSprAmTyftReURkBLqrcVqF-5zasrb3Q%40mail.gmail.com)

**Question:** How many buildfarm animals or developers run Windows with comma decimal locales, and why should the helpers stay if the frontend LC_NUMERIC fix is committed?

Counter-check: held. Two independent reports show some demand, but message #1 itself says a dot-decimal regional setting would do the job, and the thread does not weigh that against permanent helpers. [P6] is about expected files for an error message and should not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds public methods get_decimal_separator() and format_locale_number() to PostgreSQL::Test::Cluster, which extension TAP tests can call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | A buildfarm animal also runs back branches, and the proposal does not say whether the test fix goes there. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future test that passes a fractional number to psql must use the helper, and only comma-locale Windows machines show an omission. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A frontend LC_NUMERIC patch in a separate thread, or a dot-decimal regional setting, would solve the same failures without test helpers. |

## 5. Questions the author must answer before anyone reviews code

1. If the frontend LC_NUMERIC patch is committed, can these helpers be removed, and could they stay private to the affected tests until then?
2. What makes a new test that passes a plain fractional literal to psql fail on Linux CI, instead of only on a comma-locale Windows animal?
3. How were the five touched test files found, and does a full TAP run on the Russian-locale machine pass with the patch?
4. What does format_locale_number() return on a locale whose decimal separator is neither '.' nor ','?
5. How many buildfarm animals or developers run Windows with a comma decimal locale, and why is a dot-decimal regional setting not enough for them?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the TAP test framework and Windows locale behaviour of the frontends. Thomas Munro commented on the cause in message #2, but nobody has reviewed the patch.
- **Liability:** Two public test methods in Cluster.pm that extension tests may come to call, plus follow-up fixes each time a new test skips the helper. The liability lasts until a frontend fix makes the helpers unnecessary, and longer if extensions use them.
- **Author follow-up:** The author registered the CommitFest entry in August 2026 and posted again in September 2026, so the author is still active on the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 5.8 KB, changes only test files, and helps developers and buildfarm animals on comma-locale Windows. The scope of the helpers should be settled against the frontend LC_NUMERIC patch before code review.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Ongoing cost of locale-aware test helpers for one setup. Refuted (not specific): Duplicate of maintenance-fit-1, which gives the same forgotten-helper scenario under the more exact rubric item. The point about switching the machine locale is covered by demand-1.
- `standards-compat-1` Tests adapt to locale-dependent \watch parsing instead of flagging it. Refuted (already handled): psql rejects `\watch 0.5` under a comma locale today, so the patch does not cause the scenario. Messages #4 and #5 agree that the frontend behavior is a separate question, and it has its own thread.
- `already-exists-1` New Cluster.pm helpers duplicate the LC_NUMERIC=C setting in Utils.pm. Refuted (not specific): Duplicate of maintenance-fit-2, with the same scenario and the same question to the author. The duplication claim is also weak, because message #1 shows that LC_NUMERIC=C has no effect on MSVC strtod().

### Review angles

- prior-attempts: 1 objection(s). Only one objection fits this angle. P18 concerns Windows ignoring locale settings, but it was returned for lack of reproduction and answers, which does not apply here.
- standards-compat: 1 objection(s). This is a test-only change with no SQL, GUC, catalog or pg_dump surface. The one contract concern is that the shared test API encodes the current locale-dependent `\watch` parsing.
- replication-ondisk: 0 objection(s). The patch changes only TAP test files and the PostgreSQL::Test::Cluster Perl module. It adds no on-disk format, WAL record, catalog entry, wire protocol message or C-level ABI, so this angle has nothing to object to. The touched recovery test (046_checkpoint_logical_slot.pl) changes only how a \watch interval is formatted, not replication behaviour.
- security: 0 objection(s). The patch changes only TAP test files and the PostgreSQL::Test::Cluster Perl module. It adds no server or client behaviour, privilege, catalog object or exposed interface, so there is no security or privilege aspect to review. The frontend LC_NUMERIC change that could affect users is in a separate thread and out of scope here.
- maintenance-fit: 2 objection(s). Both objections are fixable. The test-only scope keeps hot-path and on-disk costs at zero.
- already-exists: 1 objection(s). Changing the machine locale is a working alternative, but buildfarm coverage of comma locales is a fair reason to fix the tests, so no objection is raised on that.
- demand: 1 objection(s). One objection only. Test-only changes cost users nothing, so the remaining concern is test maintenance for a small group that has a workaround.

### Precedents retrieved

- P1: [Mitigation for mutating locale names on Windows](https://commitfest.postgresql.org/patch/5136/) (cf, Withdrawn, 2024-08-14) - judged not relevant; reason summarised from its thread
- P2: [Revert "Fix link error for test_radixtree module on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6d9751fa8fd40c988541c9c72ac7a2095ba73c19) (revert, Reverted, 2024-03-08) - judged not relevant
- P3: [Revert "Avoid requiring Spanish locale to test NLS infrastructure."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=84a3778c79c2d28b4dc281d03ef2ab019b16483b) (revert, Reverted, 2025-12-15) - judged not relevant
- P4: [Revert "Use workaround of __builtin_setjmp only on MINGW on MSVCRT"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8f5e419484c3efb613d971ec25b9bf344db3d0b0) (revert, Reverted, 2025-04-07) - judged not relevant
- P5: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P6: [Revert test case added by commit 1e165d05fe06a9072867607886f818bc255507db.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=32ca22b02da9d8088b58b3dc64ad78464c7513a3) (revert, Reverted, 2017-08-02) - used in the review
- P7: [tsearch and tsdicts regression tests fail in Turkish locale on glibc](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (do_not_want, Not worth doing, date unknown) - used in the review
- P8: [output formfeed after every result when using the \watch command](https://commitfest.postgresql.org/patch/3576/) (cf, Returned with feedback, 2022-04-05) - judged not relevant
- P9: [TAP test to cover "EndOfLogTLI != replayTLI" case](https://commitfest.postgresql.org/patch/3427/) (cf, Rejected, 2022-01-28) - judged not relevant; reason summarised from its thread
- P10: [output formfeed after every result when using the \watch command](https://commitfest.postgresql.org/patch/3578/) (cf, Withdrawn, 2022-02-28) - judged not relevant
- P11: [Revert "Add TAP test to automate the equivalent of check_guc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52377bb81a2c0427d357bba71e7149de02d016ce) (revert, Reverted, 2022-02-12) - judged not relevant
- P12: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P13: [Track collation and ctype versions separately](https://commitfest.postgresql.org/patch/5141/) (cf, Withdrawn, 2025-02-07) - judged not relevant; reason summarised from its thread
- P14: [Allow HAVE_UCOL_STRCOLLUTF8 flag to be set conditionally on Windows](https://commitfest.postgresql.org/patch/1188/) (cf, Rejected, 2017-08-11) - judged not relevant; reason summarised from its thread
- P15: [make psql backslash commands honor "\pset null"](https://commitfest.postgresql.org/patch/4603/) (cf, Withdrawn, 2023-10-06) - judged not relevant; reason summarised from its thread
- P16: [Windows Service status is set to "SERVICE_RUNNING" when timeout is occurred (version 9.6 or earlier).](https://commitfest.postgresql.org/patch/1970/) (cf, Returned with feedback, 2019-11-25) - judged not relevant; reason summarised from its thread
- P17: [Introduce list_reverse() to make lcons() usage less inefficient](https://commitfest.postgresql.org/patch/4188/) (cf, Withdrawn, 2023-02-20) - judged not relevant; reason summarised from its thread
- P18: [lc_messages parameter doesn't work on Windows](https://commitfest.postgresql.org/patch/1518/) (cf, Returned with feedback, 2018-11-29) - used in the review; reason summarised from its thread
- P19: [Revert "Temporary patch to help debug pg_walsummary test failures."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2b520860c08ac946ec5e570065ede1c467e45618) (revert, Reverted, 2024-03-20) - judged not relevant
- P20: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P21: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P22: [Revert: Honor OID status of CREATE LIKE'd tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=764ce22af30df3d62524d41815c41b4d696f3c10) (revert, Reverted, 2015-04-26) - judged not relevant
- P23: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P24: [abs function for interval](https://commitfest.postgresql.org/patch/2345/) (cf, Returned with feedback, 2019-11-28) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 136 s; cost reported by the CLI: $1.38
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:01.579Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
