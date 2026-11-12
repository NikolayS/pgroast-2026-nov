# Design pre-review: add missed tests for gist index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7264](https://commitfest.postgresql.org/patch/7264/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: the patches are test-only and raise gistxlog.c coverage from about 18% to about 84%. The three surviving objections are fixable. already-exists-1 asks what 0002 adds beyond 027_stream_regress.pl, maintenance-fit-2 asks for the added runtime, and prior-attempts-1 asks how page recycling is made deterministic in a parallel regression group. A committer who knows GiST vacuum would want these answered before commit.

Not checked: Without the test code, nobody can tell whether gist.out prints layout-dependent values, how the test makes pages old enough to recycle, or how long the TAP test runs. The thread shows only one reported pass, on master.

## Prior attempts

- [Revert "Test that vacuum removes tuples older than OldestXmin"](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us) (related; Reverted, 2024-07-22). "This test proved to be unstable on the buildfarm, timing out before the standby could catch up on 32-bit machines where more rows were required and failing to reliably trigger multiple index vacuum rounds on 64-bit machines where fewer rows should be required." Different now: P16 tested heap tuple removal with a standby and failed on standby catch-up and row counts. This proposal tests GiST page deletion and reuse, but it also needs vacuum behaviour to be reproducible.
- [pg_checksum: add test for coverage](https://www.postgresql.org/message-id/flat/CAAcByaLPAK1e-Mpa5WUBATQsvXdiP5FXNX6+yqTrFZxg0vpngw@mail.gmail.com) (related; Returned with feedback, 2022-10-12). "As written these tests aren't providing more coverage, they run more code but they don't ensure that the produced output is correct." ([message](https://www.postgresql.org/message-id/2EFDD8CB-025E-48E0-8E69-E0D31FDF2025%40yesql.se)) Different now: P9 ran more code without checking output. This proposal compares index scans with sequential scans and uses wal_consistency_checking, so it checks results.
- [pgstattuple: add test for coverage](https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com) (related; Returned with feedback, 2022-11-22). "You definitely can't rely on such details not to change across platforms." ([message](https://www.postgresql.org/message-id/20221002071430.xfurt6znk2ruyxwy%40awork3.anarazel.de)) Different now: P7 printed platform-dependent function output. The thread does not say whether gist.out shows page counts or other layout details that differ between builds.

## Objections a reviewer is likely to raise

### 1. Recovery TAP test may duplicate replay that 027_stream_regress.pl already does

After 0001 lands, src/test/recovery/t/027_stream_regress.pl runs gist.sql on a primary with a streaming standby, so the standby replays the new page deletion and reuse records. 061_gist_crash_recovery.pl then replays the same workload a second time. Every check-world run pays for both, and a later change to the GiST workload must be made in two places.

Postgres already has 027_stream_regress.pl, which runs the main regression suite against a primary with a standby and can enable wal_consistency_checking through PG_TEST_EXTRA. The baseline in llvm-cov-raw.txt (message #1) shows gistxlog.c at about 18%, which suggests the numbers may come from a run without that test. If so, 0001 alone may already cover most of the redo lines credited to 0002. The thread does not compare the two tests or say what 0002 checks that 027 does not, such as crash restart instead of streaming replay.

**Question:** With 0001 applied and 027_stream_regress.pl included in the coverage run, which gistxlog.c lines does 061_gist_crash_recovery.pl still add?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com); `already-exists-1`</sub>

### 2. Added runtime of the recovery test and gist.sql steps is not given

Every developer and CI run of check-world pays for a new crash-restart TAP test with wal_consistency_checking, plus longer gist.sql in each make check. If many coverage patches add tests of this shape each release, suite time grows and slow buildfarm animals time out, as in [P16].

wal_consistency_checking logs full-page images and compares them on replay, which costs more than most recovery tests. No message gives timings for either patch, and no reviewer has asked. The coverage gain is real, mostly in gistxlog.c, but a committer would likely want the cost next to it.

Precedent: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us) (Reverted, 2024-07-22)

**Question:** How many seconds do 0001 and 0002 add on a typical machine and on a slow one, and is wal_consistency_checking limited to the GiST workload?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAG7L5%2B4jvJ1GyS2mpp1norqLgBWPcFKh3i3tqW4oq56sHae6FQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Page recycling in gist.sql may depend on the xmin horizon of concurrent tests

gist.sql runs in a parallel group, so another session's open snapshot can hold back the horizon. The second vacuum then leaves deleted pages unrecyclable on some buildfarm animals. Either gist.out differs and the animal goes red, or the test still passes while recycling silently goes unexercised.

A GiST page deleted by vacuum can be recycled only after its deletion XID is older than every snapshot. That depends on what else runs at the time. [P16] was reverted because a vacuum-driven test was "failing to reliably trigger multiple index vacuum rounds" across platforms. The thread does not say how 0001 makes recycling deterministic, and the only report so far is one pass on master in message #4.

Precedent: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us) (Reverted, 2024-07-22)

**Question:** How does 0001 make sure deleted pages are recyclable when other tests in the parallel group hold snapshots, and does anything detect when recycling did not happen?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CAG7L5%2B4jvJ1GyS2mpp1norqLgBWPcFKh3i3tqW4oq56sHae6FQ%40mail.gmail.com); `prior-attempts-1`</sub>

## Questions for the author

1. With 0001 applied and 027_stream_regress.pl included in the coverage run, which gistxlog.c lines does 061_gist_crash_recovery.pl still add?
2. How does 0001 make sure deleted pages are recyclable when other tests in the parallel group hold snapshots, and what fails if recycling does not happen?
3. How many seconds do 0001 and 0002 add to the test run on a typical machine and on a slow one?
4. Is wal_consistency_checking set for the whole TAP test or only for the GiST workload?
5. Are the tests meant for back branches, as the "stable" target of the CommitFest entry suggests, or for master only?

## Touch points

- **WAL / replication / standby:** 0002 adds a recovery TAP test that replays GiST page deletion and reuse records with wal_consistency_checking set.
- **backpatchability:** The CommitFest entry targets "stable", but the thread does not say whether the tests go to back branches.
- **maintenance burden ("who maintains this in 10 years"):** A new crash-restart TAP test and vacuum-dependent steps in gist.sql must stay stable on the buildfarm.
- **performance on the hot path:** see objection maintenance-fit-2: Added runtime of the recovery test and gist.sql steps is not given
- **doesn't fit what Postgres is about (extension / external tool / already solved):** 0002 may overlap with 027_stream_regress.pl, which already replays gist.sql on a standby.

## Who would commit it, and when

- Owner: silent. Someone who knows GiST vacuum, page deletion and its redo code, and who watches buildfarm stability of recovery tests.
- Priority: proceed; audience niche; review cost low. Two small test-only patches raise GiST redo coverage from about 18% to about 84% for Postgres developers. The open points are runtime, overlap with 027_stream_regress.pl, and recycling stability, and the author can answer them in the thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
