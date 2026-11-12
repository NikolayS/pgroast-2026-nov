# Design pre-review: add missed tests for gist index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7264](https://commitfest.postgresql.org/patch/7264/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [\[PATCH\] add missed tests for gist index](https://www.postgresql.org/message-id/flat/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), started 2026-09-05
- Messages read: 5 of 5
- Patch prose read: v4-0001-Test-GiST-page-recycling-and-deleting-all-entries.patch, v4-0002-Add-a-recovery-test-for-GiST-page-deletion-and-re.patch, llvm-cov-raw.txt, 0001-Test-GiST-page-recycling-and-deleting-all-entries.patch, 0002-Add-a-recovery-test-for-GiST-page-deletion-and-reuse.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: the patches are test-only and raise gistxlog.c coverage from about 18% to about 84%. The three surviving objections are fixable. already-exists-1 asks what 0002 adds beyond 027_stream_regress.pl, maintenance-fit-2 asks for the added runtime, and prior-attempts-1 asks how page recycling is made deterministic in a parallel regression group. A committer who knows GiST vacuum would want these answered before commit.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the test code, nobody can tell whether gist.out prints layout-dependent values, how the test makes pages old enough to recycle, or how long the TAP test runs. The thread shows only one reported pass, on master.

## The proposal, restated

**Problem.** The gist regression test vacuums only once after deleting rows, so emptied leaf pages are never recycled or reused, and no test deletes every entry. No test replays a GiST page deletion through recovery, and the redo side of gistxlog.c went unexecuted.

**Proposed design.** 0001 adds more vacuum, delete-all and reuse steps to gist.sql. 0002 adds a recovery TAP test that runs the same workload with wal_consistency_checking set, then crashes and restarts the server. Both tests compare an index scan against a sequential scan.

**What the author claims.**
- Line coverage of src/backend/access/gist rises from 80.3% to 87.9%.
- Coverage of gistxlog.c rises from about 18% to about 84%, and gistvacuum.c from about 59% to about 91%.
- The changes are test-only.

**Thread state.** Sho Ito reported that v3 passes the gist regression test and the new TAP test on master; the author has since posted v4 as a rebase. No committer has replied, and nobody has commented on test runtime or design.

## 2. Prior attempts

### Revert "Test that vacuum removes tuples older than OldestXmin" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575)
- **Thread:** [https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/CAAKRu_apNU2MPBK96V+bXjTq0RiZ-=A4ZTaysakpx9jxbq1dbQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAKRu_apNU2MPBK96V+bXjTq0RiZ-=A4ZTaysakpx9jxbq1dbQ@mail.gmail.com)
- **Date:** decided 2024-07-22; started 2024-07-19
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit aa607980aee08416211f003ab41aa750f5559712. This test proved to be unstable on the buildfarm, timing out before the standby could catch up on 32-bit machines where more rows were required and failing to reliably trigger multiple index vacuum rounds on 64-bit machines where fewer rows should be required. Because the instability is only known to be present on versions of Postgres with TIDStore used for dead TID storage by vacuum, this is only being reverted on master and REL_17_STABLE. As having this coverage may be valuable, there is a discussion on the thread of possible ways to stabilize the test. If that happens, a fixed test can be committed again.
- **Reason (quote):** "This test proved to be unstable on the buildfarm, timing out before the standby could catch up on 32-bit machines where more rows were required and failing to reliably trigger multiple index vacuum rounds on 64-bit machines where fewer rows should be required."
- **What is different this time:** P16 tested heap tuple removal with a standby and failed on standby catch-up and row counts. This proposal tests GiST page deletion and reuse, but it also needs vacuum behaviour to be reproducible.

### pg_checksum: add test for coverage (related)

- **Link:** [https://commitfest.postgresql.org/patch/3852/](https://commitfest.postgresql.org/patch/3852/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcByaLPAK1e-Mpa5WUBATQsvXdiP5FXNX6+yqTrFZxg0vpngw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcByaLPAK1e-Mpa5WUBATQsvXdiP5FXNX6+yqTrFZxg0vpngw@mail.gmail.com)
- **Date:** decided 2022-10-12; started 2022-08-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Daniel Gustafsson (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Daniel Gustafsson said the tests run more code but do not check that the output is correct, so they add little real coverage. The test descriptions also wrongly said --progress and --verbose enable verification, which is only the default mode. The author did not reply, and Michael Paquier returned the entry with feedback.
- **Reason (quote):** "As written these tests aren't providing more coverage, they run more code but they don't ensure that the produced output is correct." ([message](https://www.postgresql.org/message-id/2EFDD8CB-025E-48E0-8E69-E0D31FDF2025%40yesql.se))
- **What is different this time:** P9 ran more code without checking output. This proposal compares index scans with sequential scans and uses wal_consistency_checking, so it checks results.

### pgstattuple: add test for coverage (related)

- **Link:** [https://commitfest.postgresql.org/patch/3793/](https://commitfest.postgresql.org/patch/3793/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAAcByaKGq2JCBii9Z2MJY0-_phcPsseHS0NOGccGBz77k=FfjA@mail.gmail.com)
- **Date:** decided 2022-11-22; started 2022-08-02
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The tests added a dependency from pgstattuple on the bloom module, which Tom Lane objected to. The expected output also depended on platform details and failed on 32-bit systems. The author said he would fix it, but no updated version was posted, so Andres Freund marked the entry returned with feedback.
- **Reason (quote):** "You definitely can't rely on such details not to change across platforms." ([message](https://www.postgresql.org/message-id/20221002071430.xfurt6znk2ruyxwy%40awork3.anarazel.de))
- **What is different this time:** P7 printed platform-dependent function output. The thread does not say whether gist.out shows page counts or other layout details that differ between builds.


## 3. Objections a reviewer is likely to raise

### 1. Recovery TAP test may duplicate replay that 027_stream_regress.pl already does

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `already-exists-1`

After 0001 lands, src/test/recovery/t/027_stream_regress.pl runs gist.sql on a primary with a streaming standby, so the standby replays the new page deletion and reuse records. 061_gist_crash_recovery.pl then replays the same workload a second time. Every check-world run pays for both, and a later change to the GiST workload must be made in two places.

Postgres already has 027_stream_regress.pl, which runs the main regression suite against a primary with a standby and can enable wal_consistency_checking through PG_TEST_EXTRA. The baseline in llvm-cov-raw.txt (message #1) shows gistxlog.c at about 18%, which suggests the numbers may come from a run without that test. If so, 0001 alone may already cover most of the redo lines credited to 0002. The thread does not compare the two tests or say what 0002 checks that 027 does not, such as crash restart instead of streaming replay.

Thread: [#1 (shihao zhong, 2026-09-05)](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com)

**Question:** With 0001 applied and 027_stream_regress.pl included in the coverage run, which gistxlog.c lines does 061_gist_crash_recovery.pl still add?

Counter-check: held. 027_stream_regress.pl replays the main regression suite, including gist.sql, on a standby and can enable wal_consistency_checking through PG_TEST_EXTRA. The 18% baseline for gistxlog.c suggests the coverage run left that test out, and the thread does not say what 0002 adds beyond it.

### 2. Added runtime of the recovery test and gist.sql steps is not given

fixable · level 1b, rubric item 5 · maintenance burden ("who maintains this in 10 years"), performance on the hot path · `maintenance-fit-2`

Every developer and CI run of check-world pays for a new crash-restart TAP test with wal_consistency_checking, plus longer gist.sql in each make check. If many coverage patches add tests of this shape each release, suite time grows and slow buildfarm animals time out, as in [P16].

wal_consistency_checking logs full-page images and compares them on replay, which costs more than most recovery tests. No message gives timings for either patch, and no reviewer has asked. The coverage gain is real, mostly in gistxlog.c, but a committer would likely want the cost next to it.

**Precedents.**
- [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (Reverted, 2024-07-22); thread: [#1](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us), [#2](https://www.postgresql.org/message-id/flat/CAAKRu_apNU2MPBK96V+bXjTq0RiZ-=A4ZTaysakpx9jxbq1dbQ@mail.gmail.com)

Thread: [#1 (shihao zhong, 2026-09-05)](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), [#4 (Sho Ito, 2026-10-12)](https://www.postgresql.org/message-id/CAG7L5%2B4jvJ1GyS2mpp1norqLgBWPcFKh3i3tqW4oq56sHae6FQ%40mail.gmail.com)

**Question:** How many seconds do 0001 and 0002 add on a typical machine and on a slow one, and is wal_consistency_checking limited to the GiST workload?

Counter-check: held. No message gives a runtime, and a crash-restart test with wal_consistency_checking is costlier than a plain SQL test, so a committer would likely ask. [P16] timed out on standby catch-up with a large row count and must not be used as support.

### 3. Page recycling in gist.sql may depend on the xmin horizon of concurrent tests

fixable · level 2, rubric item 9 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

gist.sql runs in a parallel group, so another session's open snapshot can hold back the horizon. The second vacuum then leaves deleted pages unrecyclable on some buildfarm animals. Either gist.out differs and the animal goes red, or the test still passes while recycling silently goes unexercised.

A GiST page deleted by vacuum can be recycled only after its deletion XID is older than every snapshot. That depends on what else runs at the time. [P16] was reverted because a vacuum-driven test was "failing to reliably trigger multiple index vacuum rounds" across platforms. The thread does not say how 0001 makes recycling deterministic, and the only report so far is one pass on master in message #4.

**Precedents.**
- [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (Reverted, 2024-07-22); thread: [#1](https://www.postgresql.org/message-id/flat/614152.1721580711@sss.pgh.pa.us), [#2](https://www.postgresql.org/message-id/flat/CAAKRu_apNU2MPBK96V+bXjTq0RiZ-=A4ZTaysakpx9jxbq1dbQ@mail.gmail.com)

Thread: [#1 (shihao zhong, 2026-09-05)](https://www.postgresql.org/message-id/CAGRkXqTSFafbY-jKGapxguEiRM8ywo8naSU5ZTdbO6J3A2d37g%40mail.gmail.com), [#4 (Sho Ito, 2026-10-12)](https://www.postgresql.org/message-id/CAG7L5%2B4jvJ1GyS2mpp1norqLgBWPcFKh3i3tqW4oq56sHae6FQ%40mail.gmail.com)

**Question:** How does 0001 make sure deleted pages are recyclable when other tests in the parallel group hold snapshots, and does anything detect when recycling did not happen?

Counter-check: held. GiST recycles a deleted page only once its deletion XID precedes every snapshot, and gist.sql runs in a parallel group. The thread does not say how 0001 makes this deterministic. [P16] failed by a different mechanism, so it is only loose support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0002 adds a recovery TAP test that replays GiST page deletion and reuse records with wal_consistency_checking set. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets "stable", but the thread does not say whether the tests go to back branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new crash-restart TAP test and vacuum-dependent steps in gist.sql must stay stable on the buildfarm. |
| performance on the hot path | **yes** | see objection maintenance-fit-2: Added runtime of the recovery test and gist.sql steps is not given |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | 0002 may overlap with 027_stream_regress.pl, which already replays gist.sql on a standby. |

## 5. Questions the author must answer before anyone reviews code

1. With 0001 applied and 027_stream_regress.pl included in the coverage run, which gistxlog.c lines does 061_gist_crash_recovery.pl still add?
2. How does 0001 make sure deleted pages are recyclable when other tests in the parallel group hold snapshots, and what fails if recycling does not happen?
3. How many seconds do 0001 and 0002 add to the test run on a typical machine and on a slow one?
4. Is wal_consistency_checking set for the whole TAP test or only for the GiST workload?
5. Are the tests meant for back branches, as the "stable" target of the CommitFest entry suggests, or for master only?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows GiST vacuum, page deletion and its redo code, and who watches buildfarm stability of recovery tests.
- **Liability:** The committer owns the buildfarm stability of both tests for as long as they exist, and on each stable branch if they are back-patched.
- **Author follow-up:** The author posted three rebases (v2, v3, v4) over two months, which suggests continued attention.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Two small test-only patches raise GiST redo coverage from about 18% to about 84% for Postgres developers. The open points are runtime, overlap with 027_stream_regress.pl, and recycling stability, and the author can answer them in the thread.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Page recycling in the parallel regression schedule may not happen reliably. Refuted (not specific): Duplicate of prior-attempts-1, which quotes [P16] more accurately. [P16] was reverted for standby timeouts and TIDStore row counts, so it is not "exactly this kind" of instability.

### Review angles

- prior-attempts: 1 objection(s). The coverage-only concern from P9 does not apply, because the tests check results and redo. The remaining point is stability, raised above.
- standards-compat: 0 objection(s). The patches change only gist.sql, its expected output and a new recovery TAP test. They add no syntax, GUC, catalog object, dump content, psql behaviour or client-visible change, so nothing in the user-facing contract can break. Test stability and runtime are outside this angle.
- replication-ondisk: 0 objection(s). The patches only add tests and change no on-disk format, WAL record, catalog, protocol or extension API. Whether page recycling is deterministic under the parallel regression schedule is a test-stability question (see [P16]), and other angles cover it.
- security: 0 objection(s). The patches are test-only: they add steps to gist.sql and a new recovery TAP test, with no change to server code, privileges, roles, defaults, authentication or exposed information. There is no privilege or exposure aspect to review from the security angle.
- maintenance-fit: 2 objection(s). The proposal is test-only and fits core. Both objections concern test stability and cost and can be fixed in a new version.
- already-exists: 1 objection(s). 0001 extends the existing gist.sql test and has no alternative elsewhere. Only 0002 overlaps with an existing mechanism.
- demand: 0 objection(s). No demand objection: the patch is test-only, and its beneficiaries are Postgres developers, who need tests for GiST redo, which ran at about 18% coverage. Runtime cost and test stability belong to other angles, and the thread does not report the added runtime.

### Precedents retrieved

- P1: [Delete pages during GiST VACUUM](https://commitfest.postgresql.org/patch/1416/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P2: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P3: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant; reason summarised from its thread
- P4: [sort leaf pages by ctid for gist indexes built using sorted method](https://commitfest.postgresql.org/patch/3454/) (cf, Returned with feedback, 2022-03-22) - judged not relevant; reason summarised from its thread
- P5: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P6: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P7: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - used in the review; reason summarised from its thread
- P8: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P9: [pg_checksum: add test for coverage](https://commitfest.postgresql.org/patch/3852/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P10: [pg_waldump: add test for coverage](https://commitfest.postgresql.org/patch/3834/) (cf, Returned with feedback, 2022-09-30) - judged not relevant; reason summarised from its thread
- P11: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P12: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant
- P13: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P14: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P15: [Pass sk_attno to GIST consistent function](https://commitfest.postgresql.org/patch/4902/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P16: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - used in the review
- P17: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - used in the review
- P18: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P19: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P20: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant
- P21: [Revert "Temporary patch to help debug pg_walsummary test failures."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2b520860c08ac946ec5e570065ede1c467e45618) (revert, Reverted, 2024-03-20) - judged not relevant
- P22: [New gist vacuum](https://commitfest.postgresql.org/patch/371/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P23: [Retail IndexTuple Deletion](https://commitfest.postgresql.org/patch/1802/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P24: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 138 s; cost reported by the CLI: $1.15
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 66 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:34:11.268Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
