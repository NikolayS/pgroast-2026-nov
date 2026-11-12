# Design pre-review: [CREATE|RE] INDEX CONCURRENTLY with single heap scan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4971](https://commitfest.postgresql.org/patch/4971/) (Needs review); authors: Michail Nikolaev (nkey), Mihail Nikalayeu (mihailnikalayeu)
- Thread: [Revisiting {CREATE INDEX, REINDEX} CONCURRENTLY improvements](https://www.postgresql.org/message-id/flat/CANtu0oiLc-%2B7h9zfzOVy2cv2UuYk_5MUReVLnVbOay6OgD_KGg%40mail.gmail.com), started 2023-12-15
- Messages read: 126 of 126
- Patch prose read: v16-0009-Concurrently-built-index-validation-uses-fresh-s.patch, v16-0008-Improve-CREATE-REINDEX-INDEX-CONCURRENTLY-using-.patch, v16-0011-Add-proper-handling-of-auxiliary-indexes-during-.patch, v16-0012-Updates-index-insert-and-value-computation-logic.patch, v16-0010-Remove-PROC_IN_SAFE_IC-optimization.patch, v16-0007-tuplestore-add-support-for-storing-Datum-values.patch, v16-0005-Allow-snapshot-resets-in-concurrent-unique-index.patch, v16-0006-Add-STIR-Short-Term-Index-Replacement-access-met.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Demand is shown and reviewers support the STIR design, so nothing fails at level 1. The surviving objections are fixable level 2 points, and prior-attempts-1 is decisive: validation refreshes its snapshot, and the thread states no invariant that excludes the missing-tuple failure recorded in P5. maintenance-fit-1 and replication-ondisk-1 add two uncovered paths. A committer who knows HOT pruning and concurrent index builds has to accept the safety argument.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only v16 patch prose and 3 of 126 messages were read; v40 may already answer some points. Whether the TID merge and snapshot refresh are safe against HOT pruning cannot be judged without the code.

## The proposal, restated

**Problem.** CREATE INDEX CONCURRENTLY and REINDEX CONCURRENTLY scan the table twice and hold long-lived snapshots, so they are slow and hold back the xmin horizon. VACUUM on every table in the database then cannot remove dead tuples. An earlier fix (commit d9d076222f5b, PG14) that let VACUUM ignore such backends was reverted after a bug that left tuples missing from indexes.

**Proposed design.** Before the build, the patch creates an unlogged auxiliary index of a new STIR access method; it stores only the TIDs of tuples inserted during the build. Validation replaces the second heap scan: it merges sorted TIDs from STIR and the target index, then fetches and inserts the missing tuples. A later patch refreshes the validation snapshot every 100ms (earlier versions used every 4096 pages); snapshot resets in the first scan moved to CF entry 6401.

**User-visible surface.**
- New index access method `stir` in pg_am, with opclass and opfamily; `CREATE INDEX ... USING stir` fails with "Building STIR indexes is not supported".
- Failed builds leave an invalid auxiliary index named with suffix `_ccaux`, shown by psql `\d`; DROP INDEX on the main index drops it.
- pg_stat_progress_create_index: new phase "waiting for writers to use auxiliary index", and "index validation: scanning table" becomes "index validation: merging indexes" with tuple counts.
- C API: `index_create()` gets new parameters (relpersistence, auxiliary index link), `IndexInfo.ii_Auxiliary`, `IndexVacuumInfo.validate_index`, and Datum storage in tuplestore.
- Documentation states that concurrent builds have minimal impact on VACUUM of other tables.

**What the author claims.**
- Index builds run about 2 to 4 times faster; one pgbench test went from 117 s to 68 s.
- Concurrent pgbench throughput is not affected, and the oldest snapshot age stays near zero during the build.
- The STIR index is small (typically under 1 MiB), writes no WAL, and has no measurable insert overhead.
- Stress tests and a reviewer's banking-style consistency test found no missing tuples.
- Index AMs need no extra work to benefit.

**Thread state.** Matthias van de Meent, Heikki Linnakangas and others support the STIR approach, and logical decoding alternatives were rejected. Matthias reviewed 0002 and 0003 in August 2026 and promised a review of 0004 onward; v40 has had no reply, and CFbot reports it needs a rebase.

## 2. Prior attempts

### Revert changes to CONCURRENTLY that "sped up" Xmin advance (similar idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b)
- **Thread:** [https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)
- **Date:** decided 2022-05-31; started 2021-02-23
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit d9d076222f5b "VACUUM: ignore indexing operations with CONCURRENTLY". These changes caused indexes created with the CONCURRENTLY option to miss heap tuples that were HOT-updated and HOT-pruned during the index creation. Before these changes, HOT pruning would have been prevented by the Xmin of the transaction creating the index, but because this change was precisely to allow the Xmin to move forward ignoring that backend, now other backends scanning the table can prune them. This is not a problem for VACUUM (which requires a lock that conflicts with a CREATE INDEX CONCURRENTLY operation), but HOT-prune can definitely occur. In other words, Xmin advancement was sped up, but at the cost of corrupting the resulting index. Regrettably, this means that the new feature in PG14 that RIC/CIC on very large tables no longer force VACUUM to retain very old tuples goes away. We might try to implement it again in a later release, but for now the risk of indexes missing tuples is too high and there's no easy fix. Backpatch to 14, where this change appeared. Diagnosys-by: Andrey Borodin <x4mmm@yandex-team.ru> Diagnosys-by: Michael Paquier <michael@paquier.xyz> Diagnosys-by: Andres Freund <andres@anarazel.de>
- **Reason (quote):** "These changes caused indexes created with the CONCURRENTLY option to miss heap tuples that were HOT-updated and HOT-pruned during the index creation."
- **What is different this time:** P5 let VACUUM and pruning ignore the CIC backend's xmin. This proposal keeps a real, periodically refreshed snapshot and uses STIR to collect new TIDs in place of the second heap scan.

### avoid bloat from CREATE INDEX CONCURRENTLY (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1047/](https://commitfest.postgresql.org/patch/1047/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql)
- **Date:** decided 2017-03-07; started 2017-02-28
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane objected to the new, undocumented snapmgr API and proposed instead to make index AMs responsible for popping the snapshot. Simon Riggs answered that he had chosen the snapmgr API because an index AM that pops the snapshot wrongly would cause a crash, and he offered to follow consensus. The visible messages show no further discussion before the entry was returned with feedback.
- **Reason (quote):** "However, now that I read the patch: this is a horribly ugly hack." ([message](https://www.postgresql.org/message-id/1983.1488288643%40sss.pgh.pa.us))
- **What is different this time:** P3 released the snapshot only after the first scan, btree only, through a new snapmgr API. This proposal refreshes snapshots during validation and replaces the second scan with STIR.

### Skip all-visible pages during second HeapScan of CIC (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1057/](https://commitfest.postgresql.org/patch/1057/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CABOikdO+=3=rK_Y=8o-xd5oPiNSPsoORYThJUCNE8kWm1pWOow@mail.gmail.com](https://www.postgresql.org/message-id/flat/CABOikdO+=3=rK_Y=8o-xd5oPiNSPsoORYThJUCNE8kWm1pWOow@mail.gmail.com)
- **Date:** decided 2017-03-24; started 2017-02-28
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P13 made the second heap scan cheaper by skipping all-visible pages. This proposal removes that scan and merges STIR TIDs with the target index.


## 3. Objections a reviewer is likely to raise

### 1. Validation snapshot refresh needs a stated argument against the PG14 failure

fixable · level 2, rubric item 7 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

During validation, CIC drops its old snapshot every 100ms, so other backends HOT-prune pages behind it. If a chain root that STIR or the target index depends on is pruned between refreshes, the new index misses a live row. Users see this later as wrong query results or an amcheck failure.

The PG14 revert [P5] was caused by HOT pruning that moved forward once the CIC backend no longer held back xmin; 0007 brings back a moving xmin in the validation phase. Message #74 says resets are safe because only TIDs recorded in STIR are compared, but gives no invariant or proof. Message #3 asked for a formal argument, and the stress tests in messages #69 and #43 are evidence, not proof. Message #125 also leaves an open TODO for TIDs the build skipped.

**Precedents.**
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (Reverted, 2022-05-31); thread: [#1](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)

Thread: [#3 (Michail Nikolaev, 2023-12-17)](https://www.postgresql.org/message-id/CANtu0oizNtPUrPB0Mh%2B2vyjdijTX%3DLZvO5_dZN3%2BNqvE-CFPtw%40mail.gmail.com), [#74 (Mihail Nikalayeu, 2025-11-09)](https://www.postgresql.org/message-id/CADzfLwVaV15R2rUNZmKqLKweiN3SnUBg%3D6_qGE_ERb7cdQUD8g%40mail.gmail.com), [#125 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com)

**Question:** Which invariant ensures that every live tuple missing from the target index has its HOT-root TID in STIR, even when pruning runs between validation snapshot refreshes?

Counter-check: held. Message #74 asserts that validation resets are safe against HOT pruning but states no invariant, and the brief lists this as unexplained. P5 records the same failure class, so the precedent applies and a written argument is a fair request.

### 2. Validation correctness depends on a second, separately reviewed patch series

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

CF entry 6401 (snapshot resets in the first scan) and this series are committed in different releases. The open TODO from message #125 is not carried over. Tuples skipped by the build's duplicate-check budget never reach STIR or validation, so the finished index lacks them and nobody notices until a query returns wrong results.

Message #125 says that once the series are combined, skipped TIDs "must also reach validation, as STIR alone may miss them", and leaves a TODO for it. The completeness invariant of the index then spans two CF entries that share snapshot helpers and an interval setting. A committer of either entry must keep the other's assumptions in mind, and no test is named that fails if they diverge. The same class of bug, tuples missing from a concurrently built index, caused the PG14 revert [P5].

**Precedents.**
- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (Reverted, 2022-05-31); thread: [#1](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org), [#2](https://www.postgresql.org/message-id/flat/20210115133858.GA18931@alvherre.pgsql)

Thread: [#110 (Mihail Nikalayeu, 2026-01-15)](https://www.postgresql.org/message-id/CADzfLwVPQOF7m5x4d6%3D1tRJO%3D%3DFUZdc3LeO%3DN_TpmOzii7iH_Q%40mail.gmail.com), [#125 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com)

**Question:** Which check or test fails if 6401 lands without the skipped-TID handoff, and in which entry does that handoff live?

Counter-check: held. Message #125 says that skipped TIDs must reach validation when the two series are combined, and leaves this as a TODO. The material names no test or owning entry for that handoff, and missing index tuples is the failure class recorded in P5.

### 3. Leftover unlogged STIR index on a logged table across persistence-changing commands

fixable · level 2, rubric item 13 · on-disk format / pg_upgrade, catalog changes, WAL / replication / standby · `replication-ondisk-1`

A failed CIC leaves `idx_ccaux` as an unlogged index on a permanent table. A DBA later runs `alter table ... set unlogged` and then `set logged`, or `vacuum full`, `cluster` or `truncate`, before dropping it. These commands rebuild every index with the table's persistence, so the STIR either becomes WAL-logged against its `!RelationNeedsWAL` assertions or hits "Building STIR indexes is not supported".

Before this patch an index always had its table's persistence. Message #123 makes the STIR always unlogged on purpose and adds a `relpersistence` parameter to `index_create()`, so mixed persistence becomes a state that can persist on disk. Patch 0005 covers DROP and REINDEX of a junk `_ccaux`. The material does not say what the other rewrite paths do with it, and `_ccaux` survives until someone drops it.

Thread: [#64 (Sergey Sargsyan, 2025-06-18)](https://www.postgresql.org/message-id/CAMAof68L0GO0F0bwuXtLZAjh9k_Hj%2Bo0-8mqfO6iEQyXr4PuVA%40mail.gmail.com), [#122 (Matthias van de Meent, 2026-08-05)](https://www.postgresql.org/message-id/CAEze2WhcnPrb1F2urn1M3RAm292FXkTJ5zXQ%3DdCUyab47JJ0mw%40mail.gmail.com), [#123 (Mihail Nikalayeu, 2026-08-06)](https://www.postgresql.org/message-id/CADzfLwW%3Dq71fw0hye9K3vBnS7RR50rTcCM91ZbWkiKeU%2B5s4_g%40mail.gmail.com)

**Question:** What do SET LOGGED, SET UNLOGGED, VACUUM FULL, CLUSTER and TRUNCATE do to a leftover `_ccaux` index, and which tests cover each path?

Counter-check: held. Message #123 confirms that STIR is always unlogged, even on a permanent table. Patch 0005 covers only DROP and REINDEX, and TRUNCATE, CLUSTER and VACUUM FULL rebuild indexes through the build path, which STIR rejects.

### 4. A stub index AM in pg_am that every AM-wide change must special-case

fixable · level 2, rubric item 12 · catalog changes, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A future patch adds a new index AM callback or a new code path that loops over indexes, such as ANALYZE, VACUUM, pgstattuple or amcheck. It does not handle STIR's stub behaviour. A leftover `_ccaux` index then raises an error or an assertion, as the unclosed critical section did during ANALYZE in message #62, and users find out only in the field.

The series registers `stir` in pg_am with an opclass and opfamily, and special-cases it in analyze.c, vacuumlazy.c, vacuumparallel.c, pgstattuple and opr_sanity. Heikki in message #106 proposed a shared memory struct instead of using the catalog for this coordination. Message #107 answered with relcache-synchronization concerns, and the thread did not settle which design leaves fewer places to update. The material does not say what makes a future change fail loudly when it forgets STIR.

Thread: [#62 (Sergey Sargsyan, 2025-06-17)](https://www.postgresql.org/message-id/CAMAof69JSL8MYWG2qRScs3RQDpfcyZT_wFwW4SoAvftW%2BK_p1g%40mail.gmail.com), [#106 (Heikki Linnakangas, 2025-12-16)](https://www.postgresql.org/message-id/4b368c17-a614-401b-a335-398450249c47%40iki.fi), [#107 (Mihail Nikalayeu, 2025-12-16)](https://www.postgresql.org/message-id/CADzfLwWUM2H2%2BdaMv9q0%2BroX_NGe3cPT%2BAgvQ60Vh_J-e3ts6Q%40mail.gmail.com), [#108 (Matthias van de Meent, 2025-12-17)](https://www.postgresql.org/message-id/CAEze2WjjL2Uee7ZZzFr%2BsDe9jdFtaoX6TWdry6NRvGr2FX0ASw%40mail.gmail.com)

**Question:** What makes a new index-wide code path fail in regression tests, not in production, when it does not account for a STIR index?

Counter-check: held. The patch special-cases STIR in analyze.c, vacuumlazy.c, vacuumparallel.c and pgstattuple, and message #62 shows one such path already failed on a leftover index. Messages #107 and #108 answer the shared memory alternative but not how a forgotten special case fails in tests.

### 5. Docs promise minimal VACUUM impact before the series delivers it

fixable · level 1b, rubric item 6 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-3`

A DBA reads the new CREATE INDEX and REINDEX text that concurrent builds have minimal impact on VACUUM of other tables. The DBA then runs a long CIC on a large table. The first heap scan still holds one snapshot, because the first-scan resets now live in CF entry 6401, so the xmin horizon stays behind for the whole build and other tables bloat.

The documentation hunk comes from the full series, but message #74 removed the first-phase snapshot reset from this entry. Message #125 says the two series work independently and still leaves a TODO for skipped TIDs. If this entry is committed alone, the user-facing contract claims more than the code delivers, and that wording is hard to take back once released.

Thread: [#74 (Mihail Nikalayeu, 2025-11-09)](https://www.postgresql.org/message-id/CADzfLwVaV15R2rUNZmKqLKweiN3SnUBg%3D6_qGE_ERb7cdQUD8g%40mail.gmail.com), [#110 (Mihail Nikalayeu, 2026-01-15)](https://www.postgresql.org/message-id/CADzfLwVPQOF7m5x4d6%3D1tRJO%3D%3DFUZdc3LeO%3DN_TpmOzii7iH_Q%40mail.gmail.com), [#125 (Mihail Nikalayeu, 2026-10-17)](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com)

**Question:** What exactly will the CREATE INDEX and REINDEX docs promise about VACUUM if this entry is committed without CF entry 6401?

Counter-check: held. Message #74 moved the first-scan resets out of this entry, and the only documentation hunk shown still promises minimal impact on VACUUM. This is a wording fix and not a permanent commitment; the v40 documentation text is not in the material.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | A failed build leaves an unlogged `_ccaux` STIR index on disk, attached to a permanent table. |
| WAL / replication / standby | **yes** | The STIR index writes no WAL, but its catalog state is WAL-logged and replicated. |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds `stir` rows to pg_am, pg_opclass, pg_opfamily and pg_proc, and a new dependency link. |
| ABI / extension API | **yes** | `index_create()` gets new parameters; IndexInfo, IndexVacuumInfo, tableam.h and tuplestore change. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | psql `\d` shows a second INVALID index, `idx_ccaux` with AM `stir`, after a failed build. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new index AM is special-cased in ANALYZE, VACUUM, parallel vacuum and pgstattuple, and correctness is shared with CF entry 6401. |
| performance on the hot path | **yes** | Every insert into a table under a concurrent build also inserts a TID into the STIR index. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which invariant ensures that every live tuple missing from the target index has its HOT-root TID in STIR when pruning runs between validation snapshot refreshes?
2. Which CF entry owns the handoff of TIDs skipped by the build to validation, and which test fails if entry 6401 lands without it?
3. What do SET LOGGED, SET UNLOGGED, VACUUM FULL, CLUSTER and TRUNCATE do to a leftover unlogged `_ccaux` index on a permanent table?
4. What makes a new index-wide code path fail in regression tests when it does not account for a STIR index?
5. What will the CREATE INDEX and REINDEX documentation say about VACUUM if this entry is committed without CF entry 6401?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas supported the STIR approach in the thread, but v40 has no reply. The owner must know HOT pruning, snapshot and xmin handling, and the concurrent index build code.
- **Liability:** The owner signs for the completeness of every concurrently built index, the failure class that forced the PG14 revert (P5). The owner also carries a permanent `stir` AM in the catalog and its special cases in VACUUM and ANALYZE.
- **Author follow-up:** Strong sign: the author has rebased the series through 12 CommitFests up to v40 and answers review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Operators of large, busy tables gain faster concurrent builds and a moving xmin horizon, and several reviewers asked for this. The series is large, adds a catalog AM and touches heap, nbtree, catalog and vacuum code; it needs a rebase and a written safety argument.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Progress view phase and columns change meaning for existing monitors. Refuted (no failure scenario): The validation phase no longer scans the table, so the old phase name and block counters would report something that does not happen. The monitoring hunk documents the new phase and tuple counters; the release-note wording is outside levels 1-2.
- `standards-compat-2` Catalog-visible stir AM exposes an object users cannot recreate. Refuted (no failure scenario): pg_dump skips invalid indexes, and a tool that copies an INVALID index definition gets a clear error, which is not a data or upgrade failure. Messages #107 and #108 weighed the shared memory alternative; maintenance-fit-2 covers the remaining catalog cost.
- `replication-ondisk-2` STIR skip-insert state is lost on crash or standby promotion. Refuted (no failure scenario): After a crash or promotion the build is dead, and the result is an invalid `_ccaux` index that still receives cheap TID inserts until dropped. The documentation hunk already describes this state and its recovery, as for `_ccnew` today.
- `maintenance-fit-3` New per-insert work for all writers of a table under CIC. Refuted (already handled): Messages #43 and #46 report unchanged pgbench throughput at 8 and 50 clients while building an index on `abalance`, where every update inserts into STIR. Those runs used the single-page design, and v38 only reduces that contention; the cost falls only on tables under CIC.

### Review angles

- prior-attempts: 1 objection(s). Only one objection: the precedents mostly targeted the same goal by other means, and their recorded reasons, apart from the PG14 pruning revert, do not apply to the STIR design.
- standards-compat: 3 objection(s). The SQL syntax does not change and there are no new keywords, and pg_dump does not appear to emit invalid indexes, so the round trip is not affected. The objections above are fixable and do not reject the idea.
- replication-ondisk: 2 objection(s). WAL volume, hot standby queries, base backups and pg_dump look unaffected: STIR writes no WAL, and invalid indexes are neither used by queries nor dumped. The init-fork WAL logging issue from message #122 was fixed in message #123.
- security: 0 objection(s). The proposal changes no privilege rules. Concurrent builds still need the same ownership and locks, and `CREATE INDEX ... USING stir` is refused. Index expressions still run as the table owner, under the same restrictions as before. STIR grows only with the tuples inserted during the build, so its size follows the heap's growth, which needs no extra privilege. Not checked without the code: whether the `_ccaux` index inherits the main index's owner, tablespace and dependency entries.
- maintenance-fit: 3 objection(s). Not empty. The nbtree VACUUM-bookkeeping invariant does not apply: validation only collects TIDs and deletes no pages.
- already-exists: 0 objection(s). Postgres has no existing feature that gives a concurrent index build with one heap scan and no long-held snapshot. Plain CREATE INDEX blocks writes, and the PG14 horizon shortcut was reverted [P5]. The logical-decoding alternative was discussed and rejected in messages #89, #100, #101 and #106, and no extension or hook in the material does the job. Whether STIR should be a pg_am entry or an internal structure (message #106) is a design question for another angle.
- demand: 0 objection(s). The material shows real, lasting demand: the PG14 feature for the same goal was called "extremely valuable for users" when it was reverted [P5], and several independent people (Matthias, Heikki, Sergey, Josh, Antonin) say they want it or test it on busy systems (messages #45, #56, #69, #106, #117). The costs (a new catalog AM, an index_create() API change, two invalid indexes after a failure) are design and maintenance questions for other angles.

### Precedents retrieved

- P1: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant; reason summarised from its thread
- P2: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P3: [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (cf, Returned with feedback, 2017-03-07) - used in the review; reason summarised from its thread
- P4: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - used in the review; reason summarised from its thread
- P5: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - used in the review
- P6: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - used in the review; reason summarised from its thread
- P7: [Revert tests for drop index concurrently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5ad72cee7ed3ac0016c716f0408811274d822f14) (revert, Reverted, 2012-10-18) - judged not relevant
- P8: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P9: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [Revert "ICU: do not convert locale 'C' to 'en-US-u-va-posix'."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=455f948b0d03a556533a7e4a1a8abf45f0eb202e) (revert, Reverted, 2023-05-09) - judged not relevant
- P11: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P12: [NOT VALID for Unique Indexes](https://commitfest.postgresql.org/patch/2937/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [Skip all-visible pages during second HeapScan of CIC](https://commitfest.postgresql.org/patch/1057/) (cf, Returned with feedback, 2017-03-24) - used in the review
- P14: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - judged not relevant
- P15: [Eliminating CREATE INDEX comparator TID tie-breaker overhead](https://commitfest.postgresql.org/patch/312/) (cf, Rejected, 2015-07-30) - judged not relevant
- P16: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P17: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P18: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant
- P19: [Sequence Access Method](https://commitfest.postgresql.org/patch/44/) (cf, Returned with feedback, 2015-07-28) - judged not relevant
- P20: [brin: avoid errors processing work item following concurrent reindex](https://commitfest.postgresql.org/patch/2838/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P21: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1276/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P22: [Reloptions for table access methods](https://commitfest.postgresql.org/patch/2717/) (cf, Withdrawn, 2021-04-06) - judged not relevant
- P23: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - judged not relevant
- P24: [Sequence Access Methods](https://commitfest.postgresql.org/patch/466/) (cf, Returned with feedback, 2016-04-06) - judged not relevant

### Warnings

- attachment limit (8) reached; v16-0004-Allow-snapshot-resets-during-parallel-concurrent.patch not read
- attachment limit (8) reached; v16-0002-Add-stress-tests-for-concurrent-index-operations.patch not read
- attachment limit (8) reached; v16-0003-Allow-advancing-xmin-during-non-unique-non-paral.patch not read
- attachment limit (8) reached; v16-0001-This-is-https-commitfest.postgresql.org-50-5160-.patch not read
- attachment limit (8) reached; v1-0001-WIP-fix-d9d076222f5b-VACUUM-ignore-indexing-opera.patch not read
- 13 long messages in the middle of the thread were left out of the prompts
- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-3); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 232 s; cost reported by the CLI: $9.89
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 97 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:28:48.683Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
