# Design pre-review: amcheck: add index-all-keys-match verification for B-Tree

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6526](https://commitfest.postgresql.org/patch/6526/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [amcheck: add index-all-keys-match verification for B-Tree](https://www.postgresql.org/message-id/flat/432626F9-65DF-4F0D-B345-26CFC3E2CFAC%40yandex-team.ru), started 2026-02-17
- Messages read: 14 of 14
- Patch prose read: v7-0001-amcheck-add-indexallkeysmatch-verification-for-B-.patch, v7-0002-amcheck-detect-dangling-B-Tree-references-to-heap.patch, v7-0003-doc-describe-indexallkeysmatch-verification.patch, v1-0001-amcheck-add-indexallkeysmatch-verification-for-B-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the thread shows real demand from two independent users. Before commit, the proposal needs to name a corruption case that heapallindexed misses (already-exists-1). It also needs agreement that bt_index_check() with all options off may read heap size (maintenance-fit-2). The remaining points (maintenance-fit-1, standards-compat-2) are fixable, but only a committer who knows amcheck can sign off on the risk of false corruption reports.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the normalization coverage for the reverse direction could not be checked. Two security points are also unchecked: whether FormIndexDatum runs under the restricted security context amcheck already uses, and whether error messages print key values the caller cannot otherwise read.

## The proposal, restated

**Problem.** Existing amcheck checks miss index entries whose stored key differs from the key of the heap tuple they point to, because the index structure stays valid. The author hit such corruption with UTF-8 Russian locales around hyphens and spaces, and wants a tool to investigate it, possibly for corruption his own company caused.

**Proposed design.** A new indexallkeysmatch option makes a dedicated sequential heap scan that fingerprints visible (key, TID) pairs in a Bloom filter. The index scan probes each leaf tuple and each posting list entry, and on a Bloom miss fetches the visible heap tuple through table_fetch_tid(), following HOT chains, and compares keys with FormIndexDatum. A second patch makes both functions always report leaf TIDs beyond heap EOF, and reports unused or out-of-range heap slots only under ShareLock.

**User-visible surface.**
- New boolean parameter indexallkeysmatch on bt_index_check() and bt_index_parent_check().
- amcheck extension version bump to 1.6 with an upgrade script amcheck--1.5--1.6.sql.
- New corruption errors, for example "index tuple in index ... does not match heap tuple" and errors for index TIDs pointing to missing heap pages.
- bt_index_check() and bt_index_parent_check() now always report index TIDs beyond heap EOF, even with all options off, unless the main fork is absent.
- With both heapallindexed and indexallkeysmatch on, two Bloom filters are allocated, each up to maintenance_work_mem.

**What the author claims.**
- Detects key mismatches that heapallindexed and structural checks miss, including inside posting lists before an ordering violation shows up.
- The unconditional beyond-EOF check is the first way Postgres can notice a lost or short relation segment, and it costs nearly nothing.
- With the MVCC fetch, key-match verification is safe under AccessShareLock, so no stronger lock is needed in bt_index_check().
- Jakub Wartak measured indexallkeysmatch at about 5 times a bare bt_index_check, and about 2 times heapallindexed.
- The whole pg_amcheck test suite passes with the main-fork-absent skip.

**Thread state.** Zsolt Parragi, Wenbo Lin and Jakub Wartak reviewed, and Jakub co-authored; Wenbo's concurrent VACUUM false positive and Jakub's API, docs and standby-cache points were addressed in v7. No one has replied to v7, and the author's own TODO on bt_normalize_tuple() corner cases is only partly reported as done.

## 2. Prior attempts

### amcheck verification for GiST (related)

- **Link:** [https://commitfest.postgresql.org/patch/1800/](https://commitfest.postgresql.org/patch/1800/)
- **Thread:** [https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru)
- **Date:** decided 2019-11-28; started 2018-09-23
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Geoghegan questioned whether the checks are safe under an AccessShareLock, since Heikki's v6 had used a ShareLock. He doubted that the parent re-find logic copied from gistfinishsplit() applies here and asked how concurrent page deletion could affect it. Andrey Borodin did not answer these questions for about two months, so Michael Paquier returned the entry with feedback.
- **Reason (quote):** "My main concern now is the heavyweight lock strength needed by the new function." ([message](https://www.postgresql.org/message-id/CAH2-WznzhHKREeXs1RMFYes%2BAo7aam5dPSmvS2Hto6WUcjEk4A%40mail.gmail.com))
- **What is different this time:** P2 was a new GiST checker whose safety under AccessShareLock was not answered. This proposal uses an MVCC fetch under AccessShareLock for the key check and reports unused heap slots only under ShareLock.


## 3. Objections a reviewer is likely to raise

### 1. heapallindexed may already report the key mismatches this targets

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A DBA suspects the hyphen or space collation corruption from message #1 and runs bt_index_check with heapallindexed and indexallkeysmatch on. That costs about twice the time of heapallindexed alone, and two Bloom filters. heapallindexed alone would already have reported the corrupt index entry.

heapallindexed fingerprints each index tuple with its heap TID, and expands posting lists into one fingerprint per TID. It then probes with the key formed from each visible heap tuple. An index entry "foobar" at TID T, with "foo-bar" in the heap at T, gives a Bloom miss and an error, so the TAP test's expression swap should fail heapallindexed too. Message #1 only says that structural checks missed the corruption. The v1 commit message says heapallindexed cannot catch it but names no case, so the gain may be limited to extra wrong-key entries next to a correct one, and to clearer error messages.

Thread: [#1 (Andrey Borodin, 2026-02-17)](https://www.postgresql.org/message-id/432626F9-65DF-4F0D-B345-26CFC3E2CFAC%40yandex-team.ru), [#9 (Jakub Wartak, 2026-04-28)](https://www.postgresql.org/message-id/CAKZiRmzigRhZCunAGAu8iYyWkUdzkn0PdSyC_Ur_pMwHjCer%3DQ%40mail.gmail.com)

**Question:** Which concrete corruption, such as the Russian-locale case or the expression-swap test, passes heapallindexed but fails indexallkeysmatch?

Counter-check: held. heapallindexed fingerprints each index tuple with its TID and probes with the key formed from the visible heap tuple, so "foobar" against "foo-bar" at one TID gives an error. The thread names no case that passes heapallindexed and fails the new check.

### 2. Always-on heap size check changes a heap-free function inside an option patch

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-2`

An operator runs pg_amcheck with --exclude-table on a table known to be damaged and still checks its indexes, as the 003_check test does. With v7, bt_index_check with all options off now reads that heap's size and reports corruption whenever a higher segment is missing. The excluded table again fails the run.

Messages #9 and #10 show that pg_amcheck depends on bt_index_check not touching the heap, and v7 handles this with one special case: it skips the check only when segment 0 is absent. That rule covers the test case but leaves the behavior change in place for partly lost heaps. The change also lands in a patch that is otherwise about an opt-in option. The beyond-EOF check is cheap and independent, so it could be proposed and discussed as its own commit.

Thread: [#8 (Jakub Wartak, 2026-04-08)](https://www.postgresql.org/message-id/CAKZiRmwCT%3DpqesAjC4-2rRLWQ2uiBkrmJBhWeTgALn6yfqbuew%40mail.gmail.com), [#9 (Jakub Wartak, 2026-04-28)](https://www.postgresql.org/message-id/CAKZiRmzigRhZCunAGAu8iYyWkUdzkn0PdSyC_Ur_pMwHjCer%3DQ%40mail.gmail.com), [#10 (Andrey Borodin, 2026-07-21)](https://www.postgresql.org/message-id/470AA12B-D6CC-43D1-9030-472F4873D6FB%40yandex-team.ru), [#11 (Jakub Wartak, 2026-07-27)](https://www.postgresql.org/message-id/CAKZiRmzC8dk%3D25QNHsuVeksZkJcM6%2BGoQ7D_Owj5_dEe5fW5Ow%40mail.gmail.com), [#14 (Andrey Borodin, 2026-11-08)](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru)

**Question:** Why should bt_index_check with every option off start reading heap state, and why ship this in the indexallkeysmatch series rather than as a separate patch?

Counter-check: held. Message #10 weighs the cost and keeps the check unconditional, but the skip covers only an absent main fork. An excluded table with a lost higher segment still fails the index check, and no committer has commented on that change.

### 3. Second user of bt_normalize_tuple() widens a known source of false corruption reports

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A user runs bt_index_check with indexallkeysmatch on a table whose varlena values reach the index in a different but equivalent form. The check raises ERRCODE_INDEX_CORRUPTED "does not match heap tuple". The user reindexes or restores from backup for corruption that does not exist, and the bug surfaces only as a field report.

Message #10 says normalization has been a rich source of subtle heapallindexed bugs, and message #12 cites three follow-up fixes. Message #14 reports only that the existing TOAST and short varlena cases now run with the new option. It does not say the planned review of all known cases is complete. Each future change to datum representation must now keep two comparison directions in agreement, and a missed case shows up as a false corruption error. Reviewers of the earlier uniqueness check in [P1] called a false corruption report unacceptable.

**Precedents.**
- [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (Returned with feedback, 2021-04-08); thread: [link](https://www.postgresql.org/message-id/flat/CALT9ZEHRn5xAM5boga0qnrCmPV52bScEK2QnQ1HmUZDD301JEg@mail.gmail.com)

Thread: [#10 (Andrey Borodin, 2026-07-21)](https://www.postgresql.org/message-id/470AA12B-D6CC-43D1-9030-472F4873D6FB%40yandex-team.ru), [#12 (Andrey Borodin, 2026-07-27)](https://www.postgresql.org/message-id/5AC2156E-AC9B-4D07-AC6C-D5D5AF5DBD35%40yandex-team.ru), [#14 (Andrey Borodin, 2026-11-08)](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru)

**Question:** Which normalization cases differ between the heap-to-index and index-to-heap directions, and how does each one avoid a false corruption report?

Counter-check: held. Message #10 says the normalization corner cases apply to the new direction, and message #14 reports only the existing TOAST and short varlena tests. [P1] concerned a message that said an index only might be corrupt, so it gives weak support here.

### 4. pg_amcheck gets no option for indexallkeysmatch

fixable · level 2, rubric item 13 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A DBA who checks a cluster with pg_amcheck wants the new key-match check across all indexes. pg_amcheck has --heapallindexed, --rootdescend and --checkunique but no flag for this, so the DBA must write their own loop over bt_index_check() calls. Per-index error handling and parallelism are lost.

The touched files in all three v7 patches are under contrib/amcheck and doc/src/sgml/amcheck.sgml only, and none touches src/bin/pg_amcheck. Every earlier B-Tree option is exposed through pg_amcheck, which also has to choose call signatures by installed amcheck version. Leaving the new option out makes the feature incomplete for its main audience.

Thread: [#14 (Andrey Borodin, 2026-11-08)](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru)

**Question:** Will pg_amcheck get an option for indexallkeysmatch, and how will it handle servers with amcheck older than 1.6?

Counter-check: held. The v7 file lists contain nothing under src/bin/pg_amcheck, and the thread never discusses a pg_amcheck option. pg_amcheck exposes the earlier B-Tree options, and the review summarized in [P1] asked for the same integration.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | amcheck 1.6 changes the signatures of bt_index_check() and bt_index_parent_check(), which pg_amcheck and scripts call. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The reverse check evaluates index expressions with FormIndexDatum, and its new error messages may print key values. |
| backpatchability | **yes** | v7-0002 makes both existing functions report TIDs beyond heap EOF with every option off. |
| maintenance burden ("who maintains this in 10 years") | **yes** | bt_normalize_tuple() gets a second comparison direction, and a missed case appears as a false corruption error. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | heapallindexed may already report the key mismatch from message #1, so the added coverage needs a named case. |

## 5. Questions the author must answer before anyone reviews code

1. Which concrete corruption, such as the Russian-locale case or the expression-swap test, passes heapallindexed but fails indexallkeysmatch?
2. Why should bt_index_check() with every option off start reading heap size, and why is this in the indexallkeysmatch series instead of a separate patch?
3. What should pg_amcheck report for the indexes of an excluded table whose heap has lost a higher segment but still has segment 0?
4. Which normalization cases differ between the heap-to-index and index-to-heap directions, and how does each one avoid a false corruption report?
5. Will pg_amcheck get an option for indexallkeysmatch, and how will it handle servers with amcheck older than 1.6?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows amcheck's B-Tree verification and the heapallindexed normalization code. Álvaro Herrera is listed as reviewer on the CommitFest entry but has no message in the thread.
- **Liability:** The owner signs for every false "does not match heap tuple" report caused by a missed normalization case. The owner also signs for the changed default behaviour of bt_index_check() for as long as amcheck ships.
- **Author follow-up:** The author posted revisions through v7 in response to each review and moved the entry to the next CommitFest himself. Jakub Wartak joined as co-author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The check serves people who investigate index corruption, and its cost falls only on those who enable it, apart from the always-on heap size check. The entry has been in 4 CommitFests, v7 has no reply, and no committer has posted.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` bt_index_check() with all options off now reads heap size and can fail. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same default-behaviour change with a more concrete scenario. The scenario here is partly a true corruption report, which is not a failure.

### Review angles

- prior-attempts: 0 objection(s). No precedent covers the same idea. The one related concern, amcheck lock strength in P2, is answered in the thread. P1 was returned for inactivity and for false-positive reporting, and this thread fixed its own false-positive case in v6 and v7.
- standards-compat: 2 objection(s). The new parameter is appended to both signatures and follows the heapallindexed naming, so the SQL surface itself fits existing practice.
- replication-ondisk: 0 objection(s). The proposal is a read-only contrib/amcheck check. It adds no on-disk format, WAL record, catalog, wire protocol change or core ABI, only an extension version bump to 1.6 with an upgrade script. The one cross-node point, a stale cached heap size on a standby, was raised in message #11 and answered in message #14. A standby query whose snapshot is broken by replayed pruning is cancelled through a recovery conflict, which gives no false report.
- security: 0 objection(s). The proposal adds no new privilege path. amcheck functions are not granted to PUBLIC by default, the new option sits inside existing amcheck functions, and heapallindexed already evaluates index expressions, so the reverse check adds no new kind of code execution. The material does not show the code, so two points could not be checked. One is whether FormIndexDatum runs under the table-owner restricted security context that amcheck already uses. The other is whether the new error messages print key values that the caller could not otherwise read.
- maintenance-fit: 2 objection(s). Hot-path cost falls only on users who enable the option. For committability, no committer has posted in the thread, and the listed reviewer Álvaro Herrera is silent.
- already-exists: 1 objection(s). The unconditional beyond-EOF check has no existing equivalent in amcheck or core that I can name, so it draws no objection from this angle.
- demand: 0 objection(s). The material shows real demand. Message #1 reports production B-Tree corruption in which index keys differed from heap keys under UTF-8 Russian locales and existing amcheck checks did not catch it. Message #8 adds a second, independent need from Jakub Wartak: detecting a lost heap segment, which today shows up only as silent row loss. Other users pay little, because the costly reverse check runs only when indexallkeysmatch is set. The one always-on part is the beyond-EOF comparison, and whether that behavior change is acceptable is a design question for another angle.

### Precedents retrieved

- P1: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - used in the review; reason summarised from its thread
- P2: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P3: [Verify true root on replicas with amcheck](https://commitfest.postgresql.org/patch/2418/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P4: [Revert indexed and enlargable binary heap implementation.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=810f64a01567610af7b92b0de930f16f3e20064e) (revert, Reverted, 2024-04-11) - judged not relevant
- P5: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P6: [Add reverse collations](https://commitfest.postgresql.org/patch/2356/) (cf, Withdrawn, 2020-02-26) - judged not relevant; reason summarised from its thread
- P7: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P8: [Remove extra Sort node above a btree-compatible Index Scan](https://commitfest.postgresql.org/patch/5606/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P9: [Scan key push down to heap](https://commitfest.postgresql.org/patch/850/) (cf, Returned with feedback, 2016-12-21) - judged not relevant; reason summarised from its thread
- P10: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P11: [Add code indentation check to cirrus-ci](https://commitfest.postgresql.org/patch/4691/) (cf, Withdrawn, 2024-01-24) - judged not relevant
- P12: [hash joins with bloom filters](https://commitfest.postgresql.org/patch/1540/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [Eliminating CREATE INDEX comparator TID tie-breaker overhead](https://commitfest.postgresql.org/patch/312/) (cf, Rejected, 2015-07-30) - judged not relevant
- P15: [Introduce list_reverse() to make lcons() usage less inefficient](https://commitfest.postgresql.org/patch/4188/) (cf, Withdrawn, 2023-02-20) - judged not relevant
- P16: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P17: [Improve checking for pg_index.xmin](https://commitfest.postgresql.org/patch/2337/) (cf, Returned with feedback, 2020-03-24) - judged not relevant
- P18: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - judged not relevant
- P19: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P20: [bloom filter in hash joins with batching](https://commitfest.postgresql.org/patch/446/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P21: [Tidy fill hstv array (src/backend/access/heap/pruneheap.c)](https://commitfest.postgresql.org/patch/4734/) (cf, Returned with feedback, 2024-03-04) - judged not relevant
- P22: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P23: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant
- P24: [Tid range scan](https://commitfest.postgresql.org/patch/1755/) (cf, Returned with feedback, 2019-12-01) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 154 s; cost reported by the CLI: $2.74
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 75 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:54:17.200Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
