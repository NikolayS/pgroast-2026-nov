# Design pre-review: contrib/fuzzystrmatch: improve test coverage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7575](https://commitfest.postgresql.org/patch/7575/); [thread](https://www.postgresql.org/message-id/flat/CAJ7c6TMyroie8iW%3DAHy3n%2BU2GoNO9Pr%2BC0XFNU6Qrgz%3D6JiaKA%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patch adds queries and expected output to the existing fuzzystrmatch regression test and changes no C code, so it adds no user-visible surface and no cost for users. The remaining points are test-level: whether the expected output depends on encoding or platform, and how much run time the tests add. Confidence is medium because only one message is visible and the test content was not read.

Not checked: The test queries and expected output were not read. Encoding or platform dependence of the output, the run time added, and whether the queries check meaningful behaviour could not be judged.

## Prior attempts

- [pgstattuple: add test for coverage](https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com) (similar idea; Returned with feedback, 2022-11-22). "You definitely can't rely on such details not to change across platforms." ([message](https://www.postgresql.org/message-id/20221002071430.xfurt6znk2ruyxwy%40awork3.anarazel.de)) Different now: P8 made pgstattuple depend on bloom and checked full output that varied by platform. This patch only edits the module's own sql and expected files; whether its output is platform-independent still needs a check in the patch.
- [pg_waldump: add test for coverage](https://www.postgresql.org/message-id/flat/CAAcByaKM7zyJSDkPWv6_+apupY4fXXM3q3SRXas9MMNVPUGcsQ@mail.gmail.com) (similar idea; Returned with feedback, 2022-09-30). "Unfortunately the test doesn't seem to pass on windows, and hasn't ever done so:" ([message](https://www.postgresql.org/message-id/20220922151610.azlyasrrgf7hnmta%40awork3.anarazel.de)) Different now: P11 was a new TAP test that failed on Windows and was missing from meson.build. This patch extends an existing SQL regression test that is already registered.
- [Reloption tests improvement. Test resetting illegal option that was actually set for some reason](https://www.postgresql.org/message-id/flat/4563991.km65PDbjlG@thinkpad-pgpro) (related; Returned with feedback, 2022-08-02). "[dev hat] Specifically I don't think this patch is reviewable alone; it needs to be grouped with the functionality change that needed the additional coverage." ([message](https://www.postgresql.org/message-id/02abb1b9-cbbb-c62d-ed05-4db90397fc88%40timescale.com)) Different now: P6 tested undocumented reloption behaviour on its own. The fuzzystrmatch functions are documented and user-facing, so tests of their output have value without a related code change.
- [xml2: add test for coverage](https://www.postgresql.org/message-id/flat/CAAcByaKUg2EkDGwh_OPcVh1BPnKb3=+fN0yjSx=MaUxzs+zVGw@mail.gmail.com) (similar idea; Returned with feedback, 2023-01-31). "There has been no updates on this thread for some time, so this has been switched as Returned with Feedback." ([message](https://www.postgresql.org/message-id/CALDaNm2nJU6_Mp5xMRnaSSBEZc6FvabfmE%2BizAiKHofAg6-5UA%40mail.gmail.com)) Different now: P7 was a coverage test for contrib/xml2 that was closed because its author stopped responding. That is not a reason against the idea.
- [vacuumlo: add test to vacuumlo for test coverage](https://www.postgresql.org/message-id/flat/CAAcBya+fQ1RCoPi5UmrrCTbi_tjWs0Xgw0pbmp3bgAAfYjfKyg@mail.gmail.com) (similar idea; Returned with feedback, 2023-01-31). "There has been no updates on this thread for some time, so this has been switched as Returned with Feedback." ([message](https://www.postgresql.org/message-id/CALDaNm08LUE%2BgCb3ffdWWwev%2BcKrO8mUA6GF9iPUvDi%2BxvueHA%40mail.gmail.com)) Different now: P1 was a new TAP test for vacuumlo, returned after review requests went unanswered. That is not a reason against coverage tests.
- [Increasing regression test coverage based on GCOV analysis](https://www.postgresql.org/message-id/flat/20190822174636998766188@highgo.ca) (similar idea; Withdrawn, 2019-08-24). The visible messages do not show why. The two replies welcomed the work and asked for a commitfest entry, which the author then created, and no objection appears. (AI summary of the thread) Different now: P3 added GCOV-driven tests in core and was withdrawn with no recorded objection. It gives no reason that applies here.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do any of the new queries use non-ASCII input, and if so, are they in the encoding-guarded test file instead of fuzzystrmatch.sql?
2. Is the new expected output the same on all platforms and server encodings that the buildfarm runs, including 32-bit builds?
3. How much do the new queries add to the run time of the fuzzystrmatch regression test?
4. Which functions and edge cases do the new queries cover, and do they check documented behaviour or only reach uncovered lines?
5. Which lines of dmetaphone.c and fuzzystrmatch.c stay uncovered, and why?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch enlarges contrib/fuzzystrmatch/expected/fuzzystrmatch.out, which later changes to the module must keep current.

## Who would commit it, and when

- Owner: silent. Any committer who is comfortable with contrib regression tests and the buildfarm could take it; no deep knowledge of the module is needed. None has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. The patch touches two test files in one contrib module, about 16 KiB, and helps only developers who change fuzzystrmatch. The claimed gain is large for dmetaphone.c (17.8% to 94.9% line coverage), and the entry is tagged Good First Review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
