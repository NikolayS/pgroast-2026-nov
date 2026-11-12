# Design pre-review: amcheck: add index-all-keys-match verification for B-Tree

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6526](https://commitfest.postgresql.org/patch/6526/); [thread](https://www.postgresql.org/message-id/flat/432626F9-65DF-4F0D-B345-26CFC3E2CFAC%40yandex-team.ru), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the thread shows real demand from two independent users. Before commit, the proposal needs to name a corruption case that heapallindexed misses (already-exists-1). It also needs agreement that bt_index_check() with all options off may read heap size (maintenance-fit-2). The remaining points (maintenance-fit-1, standards-compat-2) are fixable, but only a committer who knows amcheck can sign off on the risk of false corruption reports.

Not checked: Without the code, the normalization coverage for the reverse direction could not be checked. Two security points are also unchecked: whether FormIndexDatum runs under the restricted security context amcheck already uses, and whether error messages print key values the caller cannot otherwise read.

## Prior attempts

- [amcheck verification for GiST](https://www.postgresql.org/message-id/flat/59D0DA6B-1652-4D44-B0EF-A582D5824F83@yandex-team.ru) (related; Returned with feedback, 2019-11-28). "My main concern now is the heavyweight lock strength needed by the new function." ([message](https://www.postgresql.org/message-id/CAH2-WznzhHKREeXs1RMFYes%2BAo7aam5dPSmvS2Hto6WUcjEk4A%40mail.gmail.com)) Different now: P2 was a new GiST checker whose safety under AccessShareLock was not answered. This proposal uses an MVCC fetch under AccessShareLock for the key check and reports unused heap slots only under ShareLock.

## Objections a reviewer is likely to raise

### 1. heapallindexed may already report the key mismatches this targets

A DBA suspects the hyphen or space collation corruption from message #1 and runs bt_index_check with heapallindexed and indexallkeysmatch on. That costs about twice the time of heapallindexed alone, and two Bloom filters. heapallindexed alone would already have reported the corrupt index entry.

heapallindexed fingerprints each index tuple with its heap TID, and expands posting lists into one fingerprint per TID. It then probes with the key formed from each visible heap tuple. An index entry "foobar" at TID T, with "foo-bar" in the heap at T, gives a Bloom miss and an error, so the TAP test's expression swap should fail heapallindexed too. Message #1 only says that structural checks missed the corruption. The v1 commit message says heapallindexed cannot catch it but names no case, so the gain may be limited to extra wrong-key entries next to a correct one, and to clearer error messages.

**Question:** Which concrete corruption, such as the Russian-locale case or the expression-swap test, passes heapallindexed but fails indexallkeysmatch?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/432626F9-65DF-4F0D-B345-26CFC3E2CFAC%40yandex-team.ru), [#9](https://www.postgresql.org/message-id/CAKZiRmzigRhZCunAGAu8iYyWkUdzkn0PdSyC_Ur_pMwHjCer%3DQ%40mail.gmail.com); `already-exists-1`</sub>

### 2. Always-on heap size check changes a heap-free function inside an option patch

An operator runs pg_amcheck with --exclude-table on a table known to be damaged and still checks its indexes, as the 003_check test does. With v7, bt_index_check with all options off now reads that heap's size and reports corruption whenever a higher segment is missing. The excluded table again fails the run.

Messages #9 and #10 show that pg_amcheck depends on bt_index_check not touching the heap, and v7 handles this with one special case: it skips the check only when segment 0 is absent. That rule covers the test case but leaves the behavior change in place for partly lost heaps. The change also lands in a patch that is otherwise about an opt-in option. The beyond-EOF check is cheap and independent, so it could be proposed and discussed as its own commit.

**Question:** Why should bt_index_check with every option off start reading heap state, and why ship this in the indexallkeysmatch series rather than as a separate patch?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/CAKZiRmwCT%3DpqesAjC4-2rRLWQ2uiBkrmJBhWeTgALn6yfqbuew%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAKZiRmzigRhZCunAGAu8iYyWkUdzkn0PdSyC_Ur_pMwHjCer%3DQ%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/470AA12B-D6CC-43D1-9030-472F4873D6FB%40yandex-team.ru), [#11](https://www.postgresql.org/message-id/CAKZiRmzC8dk%3D25QNHsuVeksZkJcM6%2BGoQ7D_Owj5_dEe5fW5Ow%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru); `maintenance-fit-2`</sub>

### 3. Second user of bt_normalize_tuple() widens a known source of false corruption reports

A user runs bt_index_check with indexallkeysmatch on a table whose varlena values reach the index in a different but equivalent form. The check raises ERRCODE_INDEX_CORRUPTED "does not match heap tuple". The user reindexes or restores from backup for corruption that does not exist, and the bug surfaces only as a field report.

Message #10 says normalization has been a rich source of subtle heapallindexed bugs, and message #12 cites three follow-up fixes. Message #14 reports only that the existing TOAST and short varlena cases now run with the new option. It does not say the planned review of all known cases is complete. Each future change to datum representation must now keep two comparison directions in agreement, and a missed case shows up as a false corruption error. Reviewers of the earlier uniqueness check in [P1] called a false corruption report unacceptable.

Precedent: [Improve amcheck to check UNIQUE constraint in btree index](https://www.postgresql.org/message-id/flat/CALT9ZEHRn5xAM5boga0qnrCmPV52bScEK2QnQ1HmUZDD301JEg@mail.gmail.com) (Returned with feedback, 2021-04-08)

**Question:** Which normalization cases differ between the heap-to-index and index-to-heap directions, and how does each one avoid a false corruption report?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#10](https://www.postgresql.org/message-id/470AA12B-D6CC-43D1-9030-472F4873D6FB%40yandex-team.ru), [#12](https://www.postgresql.org/message-id/5AC2156E-AC9B-4D07-AC6C-D5D5AF5DBD35%40yandex-team.ru), [#14](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru); `maintenance-fit-1`</sub>

### 4. pg_amcheck gets no option for indexallkeysmatch

A DBA who checks a cluster with pg_amcheck wants the new key-match check across all indexes. pg_amcheck has --heapallindexed, --rootdescend and --checkunique but no flag for this, so the DBA must write their own loop over bt_index_check() calls. Per-index error handling and parallelism are lost.

The touched files in all three v7 patches are under contrib/amcheck and doc/src/sgml/amcheck.sgml only, and none touches src/bin/pg_amcheck. Every earlier B-Tree option is exposed through pg_amcheck, which also has to choose call signatures by installed amcheck version. Leaving the new option out makes the feature incomplete for its main audience.

**Question:** Will pg_amcheck get an option for indexallkeysmatch, and how will it handle servers with amcheck older than 1.6?

<sub>fixable, level 2, design; messages [#14](https://www.postgresql.org/message-id/83FEA711-31BD-4F26-9E9C-BFDE606D9030%40yandex-team.ru); `standards-compat-2`</sub>

## Questions for the author

1. Which concrete corruption, such as the Russian-locale case or the expression-swap test, passes heapallindexed but fails indexallkeysmatch?
2. Why should bt_index_check() with every option off start reading heap size, and why is this in the indexallkeysmatch series instead of a separate patch?
3. What should pg_amcheck report for the indexes of an excluded table whose heap has lost a higher segment but still has segment 0?
4. Which normalization cases differ between the heap-to-index and index-to-heap directions, and how does each one avoid a false corruption report?
5. Will pg_amcheck get an option for indexallkeysmatch, and how will it handle servers with amcheck older than 1.6?

## Touch points

- **ABI / extension API:** amcheck 1.6 changes the signatures of bt_index_check() and bt_index_parent_check(), which pg_amcheck and scripts call.
- **security / privileges:** The reverse check evaluates index expressions with FormIndexDatum, and its new error messages may print key values.
- **backpatchability:** v7-0002 makes both existing functions report TIDs beyond heap EOF with every option off.
- **maintenance burden ("who maintains this in 10 years"):** bt_normalize_tuple() gets a second comparison direction, and a missed case appears as a false corruption error.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** heapallindexed may already report the key mismatch from message #1, so the added coverage needs a named case.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows amcheck's B-Tree verification and the heapallindexed normalization code. Álvaro Herrera is listed as reviewer on the CommitFest entry but has no message in the thread.
- Priority: needs champion; audience niche; review cost medium. The check serves people who investigate index corruption, and its cost falls only on those who enable it, apart from the always-on heap size check. The entry has been in 4 CommitFests, v7 has no reply, and no committer has posted.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
