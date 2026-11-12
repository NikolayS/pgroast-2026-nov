# Design pre-review: Windows locale and test portability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7193](https://commitfest.postgresql.org/patch/7193/); [thread](https://www.postgresql.org/message-id/flat/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the goal of passing TAP tests on comma-locale Windows has two independent reporters. The open points are fixable design questions. maintenance-fit-2 asks whether public Cluster.pm helpers should exist while a frontend fix that makes them unnecessary is under discussion. maintenance-fit-1 notes that nothing fails on common platforms when a new test skips the helper. demand-1 is weaker, because the change is test-only and costs users nothing.

Not checked: The code was not read, so the caching of the detected separator, the behaviour on Unix with a non-C LC_NUMERIC, and the handling of other separators could not be checked. Messages #4 and #5 were not shown in full.

## Prior attempts

- [tsearch and tsdicts regression tests fail in Turkish locale on glibc](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (related; Not worth doing, date unknown). reason not recorded Different now: P7 was about regression tests failing under a Turkish locale on glibc and records no reason. This patch covers TAP tests under comma-decimal locales on Windows.

## Objections a reviewer is likely to raise

### 1. Test API may become dead weight if frontend locale handling changes

The test helpers ship in a release and extension TAP suites start calling get_decimal_separator(). Later, Andrew Bille's frontend patch makes psql honor LC_NUMERIC=C on Windows, so the helpers always return '.'. Core cannot remove them without breaking the extension tests that call them.

Messages #2, #3 and #5 treat the cause as frontend locale initialization on Windows. Message #3 shows the same \watch failure for real users, and the thread still has an open fix for it. The patch adds public methods to PostgreSQL::Test::Cluster, which extension tests use, to work around behaviour that may change soon. Messages #4 and #5 agree the two fixes are independent, but they do not discuss whether the test-side API should be permanent once the frontend fix lands.

**Question:** If the frontend fix is committed, can these helpers be removed, and could the test fix be kept private to the affected tests so that extensions do not start depending on them?

<sub>fixable, level 2, ABI / extension API; messages [#2](https://www.postgresql.org/message-id/CA%2BhUKGK8Mru903e%2BETdi8-cuoMLDFcggyE4ZzSuJtu9oVaUZmg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAJnzarxD%3DFPRA14BVyStYWgr9HekapRU%2B_uDOmdNg%2BQF5McmKg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/A733EF40-6E1A-4670-BF83-5E729434340C%40yandex-team.ru), [#5](https://www.postgresql.org/message-id/CAJnzary%2BNkyfLF%2B5CQKSprAmTyftReURkBLqrcVqF-5zasrb3Q%40mail.gmail.com); `maintenance-fit-2`</sub>

### 2. New tests with fractional \watch values will silently skip the helper

A developer later adds a TAP test that runs `\watch 0.1` as a plain literal. It passes on Linux CI and on most Windows animals, then fails only on a comma-locale Windows animal, possibly weeks later. Someone without that setup has to work out the cause and retrofit format_locale_number().

The patch fixes five existing call sites by hand, so every future test that passes a fractional number to psql or parses a frontend's decimal output must remember the helper. On common platforms nothing fails when a test leaves it out. [P6] is a case where a test was removed because carrying platform-specific locale behaviour in tests 'in perpetuity' was judged not worth it. The thread does not say how the five files were found or how new cases would be caught.

Precedent: [Revert test case added by commit 1e165d05fe06a9072867607886f818bc255507db.](https://www.postgresql.org/message-id/flat/CAKKotZS-wcDcofXDCH=sidiuajE+nqHn2CGjLLX78anyDmi3gQ@mail.gmail.com) (Reverted, 2017-08-02)

**Question:** What makes a new test that passes a plain fractional literal to psql fail on Linux CI, rather than only on a comma-locale Windows animal?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru); `maintenance-fit-1`</sub>

### 3. Small affected group with a known workaround; show more demand first

A test author later adds `\watch 0.5` without format_locale_number(). Every Unix and dot-locale animal stays green. The test breaks only on a comma-locale Windows box, which the patch exists to support. That box may be one machine, and its owner can switch Windows regional settings to a dot locale.

The demand shown is two developers with Russian-locale Windows machines (messages #1, #3) and one planned buildfarm animal. Message #1 offers changing the machine locale as an alternative and calls the fix "ugly". Andrew Bille's frontend LC_NUMERIC patch (messages #3, #5) would fix both the `\watch` failure and pg_test_timing output, which would make the new Cluster.pm helpers unnecessary. [P6] records the same judgement: carrying test machinery for Windows locale variation was not worth it once the code path worked.

Precedent: [Revert test case added by commit 1e165d05fe06a9072867607886f818bc255507db.](https://www.postgresql.org/message-id/flat/CAKKotZS-wcDcofXDCH=sidiuajE+nqHn2CGjLLX78anyDmi3gQ@mail.gmail.com) (Reverted, 2017-08-02)

**Question:** How many buildfarm animals or developers run Windows with comma decimal locales, and why should the helpers stay if the frontend LC_NUMERIC fix is committed?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/9247F741-DDAB-4005-93D2-9C6D4F3495C3%40yandex-team.ru), [#3](https://www.postgresql.org/message-id/CAJnzarxD%3DFPRA14BVyStYWgr9HekapRU%2B_uDOmdNg%2BQF5McmKg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAJnzary%2BNkyfLF%2B5CQKSprAmTyftReURkBLqrcVqF-5zasrb3Q%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. If the frontend LC_NUMERIC patch is committed, can these helpers be removed, and could they stay private to the affected tests until then?
2. What makes a new test that passes a plain fractional literal to psql fail on Linux CI, instead of only on a comma-locale Windows animal?
3. How were the five touched test files found, and does a full TAP run on the Russian-locale machine pass with the patch?
4. What does format_locale_number() return on a locale whose decimal separator is neither '.' nor ','?
5. How many buildfarm animals or developers run Windows with a comma decimal locale, and why is a dot-decimal regional setting not enough for them?

## Touch points

- **ABI / extension API:** The patch adds public methods get_decimal_separator() and format_locale_number() to PostgreSQL::Test::Cluster, which extension TAP tests can call.
- **backpatchability:** A buildfarm animal also runs back branches, and the proposal does not say whether the test fix goes there.
- **maintenance burden ("who maintains this in 10 years"):** Every future test that passes a fractional number to psql must use the helper, and only comma-locale Windows machines show an omission.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A frontend LC_NUMERIC patch in a separate thread, or a dot-decimal regional setting, would solve the same failures without test helpers.

## Who would commit it, and when

- Owner: silent. Someone who knows the TAP test framework and Windows locale behaviour of the frontends. Thomas Munro commented on the cause in message #2, but nobody has reviewed the patch.
- Priority: fix design first; audience niche; review cost low. The patch is 5.8 KB, changes only test files, and helps developers and buildfarm animals on comma-locale Windows. The scope of the helpers should be settled against the frontend LC_NUMERIC patch before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
