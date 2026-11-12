# Design pre-review: contrib/fuzzystrmatch: improve test coverage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7575](https://commitfest.postgresql.org/patch/7575/) (Needs review); authors: Aleksander Alekseev (a.alekseev)
- Thread: [\[PATCH\] contrib/fuzzystrmatch: improve test coverage](https://www.postgresql.org/message-id/flat/CAJ7c6TMyroie8iW%3DAHy3n%2BU2GoNO9Pr%2BC0XFNU6Qrgz%3D6JiaKA%40mail.gmail.com), started 2026-11-02
- Messages read: 1 of 1
- Patch prose read: v1-0001-contrib-fuzzystrmatch-imporve-test-coverage.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patch adds queries and expected output to the existing fuzzystrmatch regression test and changes no C code, so it adds no user-visible surface and no cost for users. The remaining points are test-level: whether the expected output depends on encoding or platform, and how much run time the tests add. Confidence is medium because only one message is visible and the test content was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The test queries and expected output were not read. Encoding or platform dependence of the output, the run time added, and whether the queries check meaningful behaviour could not be judged.

## The proposal, restated

**Problem.** The author says contrib/fuzzystrmatch has poor test coverage, mainly in dmetaphone.c and fuzzystrmatch.c. This matters to developers who change the module.

**Proposed design.** The patch adds test queries to the existing fuzzystrmatch regression test and updates its expected output. It changes no C code, documentation or behaviour.

**What the author claims.**
- Line coverage of dmetaphone.c rises from 17.8% to 94.9%.
- Line coverage of fuzzystrmatch.c rises from 49.5% to 98.9%.
- Line coverage of daitch_mokotoff.c rises from 95.7% to 98.8%.
- Coverage measurement works best with lcov 1.16 or older.

**Thread state.** Only the first message is visible. No one has replied yet.

## 2. Prior attempts

### pgstattuple: add test for coverage (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3793/](https://commitfest.postgresql.org/patch/3793/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com)
- **Date:** decided 2022-11-22; started 2022-08-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The tests added a dependency from pgstattuple on the bloom module, which Tom Lane objected to. The expected output also depended on platform details and failed on 32-bit systems. The author said he would fix it, but no updated version was posted, so Andres Freund marked the entry returned with feedback.
- **Reason (quote):** "You definitely can't rely on such details not to change across platforms." ([message](https://www.postgresql.org/message-id/20221002071430.xfurt6znk2ruyxwy%40awork3.anarazel.de))
- **What is different this time:** P8 made pgstattuple depend on bloom and checked full output that varied by platform. This patch only edits the module's own sql and expected files; whether its output is platform-independent still needs a check in the patch.

### pg_waldump: add test for coverage (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3834/](https://commitfest.postgresql.org/patch/3834/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcByaKM7zyJSDkPWv6_+apupY4fXXM3q3SRXas9MMNVPUGcsQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcByaKM7zyJSDkPWv6_+apupY4fXXM3q3SRXas9MMNVPUGcsQ@mail.gmail.com)
- **Date:** decided 2022-09-30; started 2022-08-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund, Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The test never passed on Windows in the cfbot runs, and the patch did not add the test to meson.build after the meson merge. Earlier review asked for test names, use of command_like(), checks of the actual output, and options placed before non-option arguments. The visible messages show no reply from the author after these reviews.
- **Reason (quote):** "Unfortunately the test doesn't seem to pass on windows, and hasn't ever done so:" ([message](https://www.postgresql.org/message-id/20220922151610.azlyasrrgf7hnmta%40awork3.anarazel.de))
- **What is different this time:** P11 was a new TAP test that failed on Windows and was missing from meson.build. This patch extends an existing SQL regression test that is already registered.

### Reloption tests improvement. Test resetting illegal option that was actually set for some reason (related)

- **Link:** [https://commitfest.postgresql.org/patch/3747/](https://commitfest.postgresql.org/patch/3747/)
- **Thread:** [https://www.postgresql.org/message-id/flat/4563991.km65PDbjlG@thinkpad-pgpro](https://www.postgresql.org/message-id/flat/4563991.km65PDbjlG@thinkpad-pgpro)
- **Date:** decided 2022-08-02; started 2022-02-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Greg Stark, Jacob Champion (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Greg Stark judged the test not worth review time, because it covers undocumented behaviour and mostly checks one specific implementation. The re-registered entry then went through another commitfest without review. Jacob Champion closed it and said the test should be posted with the functionality change that needs the coverage, so reviewers can judge its value.
- **Reason (quote):** "[dev hat] Specifically I don't think this patch is reviewable alone; it needs to be grouped with the functionality change that needed the additional coverage." ([message](https://www.postgresql.org/message-id/02abb1b9-cbbb-c62d-ed05-4db90397fc88%40timescale.com))
- **What is different this time:** P6 tested undocumented reloption behaviour on its own. The fuzzystrmatch functions are documented and user-facing, so tests of their output have value without a related code change.

### xml2: add test for coverage (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3833/](https://commitfest.postgresql.org/patch/3833/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcByaKUg2EkDGwh_OPcVh1BPnKb3=+fN0yjSx=MaUxzs+zVGw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcByaKUg2EkDGwh_OPcVh1BPnKb3=+fN0yjSx=MaUxzs+zVGw@mail.gmail.com)
- **Date:** decided 2023-01-31; started 2022-08-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author did not respond after Peter Eisentraut said the author should provide a patch for the missing tests and documentation and asked whether work was continuing. A follow-up from vignesh C also got no reply, so the entry was closed for inactivity.
- **Reason (quote):** "There has been no updates on this thread for some time, so this has been switched as Returned with Feedback." ([message](https://www.postgresql.org/message-id/CALDaNm2nJU6_Mp5xMRnaSSBEZc6FvabfmE%2BizAiKHofAg6-5UA%40mail.gmail.com))
- **What is different this time:** P7 was a coverage test for contrib/xml2 that was closed because its author stopped responding. That is not a reason against the idea.

### vacuumlo: add test to vacuumlo for test coverage (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3877/](https://commitfest.postgresql.org/patch/3877/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcBya+fQ1RCoPi5UmrrCTbi_tjWs0Xgw0pbmp3bgAAfYjfKyg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcBya+fQ1RCoPi5UmrrCTbi_tjWs0Xgw0pbmp3bgAAfYjfKyg@mail.gmail.com)
- **Date:** decided 2023-01-31; started 2022-09-03
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author did not send a revised patch after review, so the entry was returned with feedback. Daniel Gustafsson asked for more scenarios, such as a non-existing database or one with no large objects, and asked the test to use run_command(). Ian Lawrence Barwick noted that the test was not added to meson.build.
- **Reason (quote):** "There has been no updates on this thread for some time, so this has been switched as Returned with Feedback." ([message](https://www.postgresql.org/message-id/CALDaNm08LUE%2BgCb3ffdWWwev%2BcKrO8mUA6GF9iPUvDi%2BxvueHA%40mail.gmail.com))
- **What is different this time:** P1 was a new TAP test for vacuumlo, returned after review requests went unanswered. That is not a reason against coverage tests.

### Increasing regression test coverage based on GCOV analysis (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2257/](https://commitfest.postgresql.org/patch/2257/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20190822174636998766188@highgo.ca](https://www.postgresql.org/message-id/flat/20190822174636998766188@highgo.ca)
- **Date:** decided 2019-08-24; started 2019-08-22
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The two replies welcomed the work and asked for a commitfest entry, which the author then created, and no objection appears.
- **Reason (quote):** quote unavailable
- **What is different this time:** P3 added GCOV-driven tests in core and was withdrawn with no recorded objection. It gives no reason that applies here.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch enlarges contrib/fuzzystrmatch/expected/fuzzystrmatch.out, which later changes to the module must keep current. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do any of the new queries use non-ASCII input, and if so, are they in the encoding-guarded test file instead of fuzzystrmatch.sql?
2. Is the new expected output the same on all platforms and server encodings that the buildfarm runs, including 32-bit builds?
3. How much do the new queries add to the run time of the fuzzystrmatch regression test?
4. Which functions and edge cases do the new queries cover, and do they check documented behaviour or only reach uncovered lines?
5. Which lines of dmetaphone.c and fuzzystrmatch.c stay uncovered, and why?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Any committer who is comfortable with contrib regression tests and the buildfarm could take it; no deep knowledge of the module is needed. None has replied in the visible thread.
- **Liability:** The committer signs for buildfarm stability of the new expected output on all platforms and encodings. After that, the cost is keeping one expected file current when the module changes.
- **Author follow-up:** Message #1 says "Feedback is most welcome"; there is no other sign either way, and the commit message still has TODO placeholders.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches two test files in one contrib module, about 16 KiB, and helps only developers who change fuzzystrmatch. The claimed gain is large for dmetaphone.c (17.8% to 94.9% line coverage), and the entry is tagged Good First Review.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The precedents are coverage-test patches. Most died from author inactivity, which is not a reason against the idea. P8's platform-dependent output and P11's Windows failure do not carry over to deterministic SQL tests in an existing file. One point needs checking in the patch: any non-ASCII input should stay in the encoding-guarded test, because the main test file has no encoding guard.
- standards-compat: 0 objection(s). The patch only adds queries and expected output to the existing fuzzystrmatch regression test. It adds no syntax, GUC, catalog object, psql behaviour or dump content, and does not change behaviour for existing users, so the standards and compatibility angle has nothing to object to. Whether the expected output depends on encoding or platform, as in [P8], is a test-level question outside this angle.
- replication-ondisk: 0 objection(s). The patch changes only the fuzzystrmatch regression test and its expected output. It touches no on-disk format, WAL, catalog, wire protocol or extension API, so this angle has nothing to object to.
- security: 0 objection(s). The patch only adds test queries and expected output to the fuzzystrmatch regression test. It adds no new function, privilege, default, or code path that users can reach, so there is nothing to object to on security or privilege grounds.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle, a test-only patch to one contrib module adds no hot-path cost, no API and no lasting commitment beyond keeping one expected output file current. The coverage-test precedents ([P1], [P7], [P8], [P11]) were mostly closed for author inactivity or for platform-dependent output, which is a code-level matter. Two points need a look at the code: whether dmetaphone tests on non-ASCII input make the expected output depend on encoding, and how much the tests add to run time.
- already-exists: 0 objection(s). The patch only adds regression tests to an existing contrib test file. No existing feature, extension or external tool already provides this test coverage, and the patch adds no second mechanism, so this angle has nothing to raise. Platform or encoding dependence of the expected output is a different angle.
- demand: 0 objection(s). The demand angle has no fair objection here. The patch changes no behaviour and adds no user surface; its only users are developers, and dmetaphone.c at 17.8% line coverage is a real gap in documented, user-facing functions. The [P6] reason, that a test belongs with the change that needs it, was about undocumented behaviour and does not carry over. Run time and platform-dependent output belong to other angles.

### Precedents retrieved

- P1: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P3: [Increasing regression test coverage based on GCOV analysis](https://commitfest.postgresql.org/patch/2257/) (cf, Withdrawn, 2019-08-24) - used in the review; reason summarised from its thread
- P4: [Improving test coverage of extensions with pg_dump](https://commitfest.postgresql.org/patch/187/) (cf, Returned with feedback, 2015-12-02) - judged not relevant
- P5: [psql - improve code coverage](https://commitfest.postgresql.org/patch/2262/) (cf, Returned with feedback, 2020-07-30) - used in the review; reason summarised from its thread
- P6: [Reloption tests improvement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3747/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P7: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P8: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - used in the review; reason summarised from its thread
- P9: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P10: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant; reason summarised from its thread
- P11: [pg_waldump: add test for coverage](https://commitfest.postgresql.org/patch/3834/) (cf, Returned with feedback, 2022-09-30) - used in the review; reason summarised from its thread
- P12: [Rewriting the test of pg_upgrade as a TAP test - take three](https://commitfest.postgresql.org/patch/3120/) (cf, Withdrawn, 2021-11-18) - judged not relevant
- P13: [Rewriting the test of pg_upgrade as a TAP test - take two](https://commitfest.postgresql.org/patch/1497/) (cf, Returned with feedback, 2018-04-07) - judged not relevant
- P14: [pg_checksum: add test for coverage](https://commitfest.postgresql.org/patch/3852/) (cf, Returned with feedback, 2022-10-12) - used in the review
- P15: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3534/) (cf, Returned with feedback, 2022-06-29) - judged not relevant
- P16: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - used in the review
- P17: [Reloption tests iprovement. Test resetting illegal option that was actually set for some reason](https://commitfest.postgresql.org/patch/3731/) (cf, Withdrawn, 2022-07-01) - judged not relevant
- P18: [Revert "Rename contrib module basic_archive to basic_wal_module"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9aeff092c05820d6a9252d7a3f8db86f71f2205f) (revert, Reverted, 2023-01-26) - judged not relevant
- P19: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P20: [Anyarray contrib module](https://commitfest.postgresql.org/patch/145/) (cf, Rejected, 2015-04-08) - judged not relevant
- P21: [Regression tests for V2 protocol.](https://commitfest.postgresql.org/patch/1303/) (cf, Returned with feedback, 2017-11-27) - judged not relevant
- P22: [pg_background contrib module proposal](https://commitfest.postgresql.org/patch/893/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P23: [Detect test suite sideeffects in pg_regress](https://commitfest.postgresql.org/patch/4321/) (cf, Withdrawn, 2024-04-03) - judged not relevant
- P24: [Add test module for Table Access Method](https://commitfest.postgresql.org/patch/4588/) (cf, Returned with feedback, 2024-01-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 118 s; cost reported by the CLI: $0.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 9 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:31.003Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
