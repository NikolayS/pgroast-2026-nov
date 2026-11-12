# Design pre-review: [CREATE|RE] INDEX CONCURRENTLY with single heap scan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4971](https://commitfest.postgresql.org/patch/4971/); [thread](https://www.postgresql.org/message-id/flat/CANtu0oiLc-%2B7h9zfzOVy2cv2UuYk_5MUReVLnVbOay6OgD_KGg%40mail.gmail.com), 126 of 126 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Demand is shown and reviewers support the STIR design, so nothing fails at level 1. The surviving objections are fixable level 2 points, and prior-attempts-1 is decisive: validation refreshes its snapshot, and the thread states no invariant that excludes the missing-tuple failure recorded in P5. maintenance-fit-1 and replication-ondisk-1 add two uncovered paths. A committer who knows HOT pruning and concurrent index builds has to accept the safety argument.

Not checked: Only v16 patch prose and 3 of 126 messages were read; v40 may already answer some points. Whether the TID merge and snapshot refresh are safe against HOT pruning cannot be judged without the code.

## Prior attempts

- [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (similar idea; Reverted, 2022-05-31). "These changes caused indexes created with the CONCURRENTLY option to miss heap tuples that were HOT-updated and HOT-pruned during the index creation." Different now: P5 let VACUUM and pruning ignore the CIC backend's xmin. This proposal keeps a real, periodically refreshed snapshot and uses STIR to collect new TIDs in place of the second heap scan.
- [avoid bloat from CREATE INDEX CONCURRENTLY](https://www.postgresql.org/message-id/flat/20170228052302.r7nvt4sbnsgu6job@alvherre.pgsql) (similar idea; Returned with feedback, 2017-03-07). "However, now that I read the patch: this is a horribly ugly hack." ([message](https://www.postgresql.org/message-id/1983.1488288643%40sss.pgh.pa.us)) Different now: P3 released the snapshot only after the first scan, btree only, through a new snapmgr API. This proposal refreshes snapshots during validation and replaces the second scan with STIR.
- [Skip all-visible pages during second HeapScan of CIC](https://www.postgresql.org/message-id/flat/CABOikdO+=3=rK_Y=8o-xd5oPiNSPsoORYThJUCNE8kWm1pWOow@mail.gmail.com) (similar idea; Returned with feedback, 2017-03-24). reason not recorded Different now: P13 made the second heap scan cheaper by skipping all-visible pages. This proposal removes that scan and merges STIR TIDs with the target index.

## Objections a reviewer is likely to raise

### 1. Validation snapshot refresh needs a stated argument against the PG14 failure

During validation, CIC drops its old snapshot every 100ms, so other backends HOT-prune pages behind it. If a chain root that STIR or the target index depends on is pruned between refreshes, the new index misses a live row. Users see this later as wrong query results or an amcheck failure.

The PG14 revert [P5] was caused by HOT pruning that moved forward once the CIC backend no longer held back xmin; 0007 brings back a moving xmin in the validation phase. Message #74 says resets are safe because only TIDs recorded in STIR are compared, but gives no invariant or proof. Message #3 asked for a formal argument, and the stress tests in messages #69 and #43 are evidence, not proof. Message #125 also leaves an open TODO for TIDs the build skipped.

Precedent: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (Reverted, 2022-05-31)

**Question:** Which invariant ensures that every live tuple missing from the target index has its HOT-root TID in STIR, even when pruning runs between validation snapshot refreshes?

<sub>fixable, level 2, prior rejection; messages [#3](https://www.postgresql.org/message-id/CANtu0oizNtPUrPB0Mh%2B2vyjdijTX%3DLZvO5_dZN3%2BNqvE-CFPtw%40mail.gmail.com), [#74](https://www.postgresql.org/message-id/CADzfLwVaV15R2rUNZmKqLKweiN3SnUBg%3D6_qGE_ERb7cdQUD8g%40mail.gmail.com), [#125](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Validation correctness depends on a second, separately reviewed patch series

CF entry 6401 (snapshot resets in the first scan) and this series are committed in different releases. The open TODO from message #125 is not carried over. Tuples skipped by the build's duplicate-check budget never reach STIR or validation, so the finished index lacks them and nobody notices until a query returns wrong results.

Message #125 says that once the series are combined, skipped TIDs "must also reach validation, as STIR alone may miss them", and leaves a TODO for it. The completeness invariant of the index then spans two CF entries that share snapshot helpers and an interval setting. A committer of either entry must keep the other's assumptions in mind, and no test is named that fails if they diverge. The same class of bug, tuples missing from a concurrently built index, caused the PG14 revert [P5].

Precedent: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://www.postgresql.org/message-id/flat/17485-396609c6925b982d@postgresql.org) (Reverted, 2022-05-31)

**Question:** Which check or test fails if 6401 lands without the skipped-TID handoff, and in which entry does that handoff live?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#110](https://www.postgresql.org/message-id/CADzfLwVPQOF7m5x4d6%3D1tRJO%3D%3DFUZdc3LeO%3DN_TpmOzii7iH_Q%40mail.gmail.com), [#125](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Leftover unlogged STIR index on a logged table across persistence-changing commands

A failed CIC leaves `idx_ccaux` as an unlogged index on a permanent table. A DBA later runs `alter table ... set unlogged` and then `set logged`, or `vacuum full`, `cluster` or `truncate`, before dropping it. These commands rebuild every index with the table's persistence, so the STIR either becomes WAL-logged against its `!RelationNeedsWAL` assertions or hits "Building STIR indexes is not supported".

Before this patch an index always had its table's persistence. Message #123 makes the STIR always unlogged on purpose and adds a `relpersistence` parameter to `index_create()`, so mixed persistence becomes a state that can persist on disk. Patch 0005 covers DROP and REINDEX of a junk `_ccaux`. The material does not say what the other rewrite paths do with it, and `_ccaux` survives until someone drops it.

**Question:** What do SET LOGGED, SET UNLOGGED, VACUUM FULL, CLUSTER and TRUNCATE do to a leftover `_ccaux` index, and which tests cover each path?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#64](https://www.postgresql.org/message-id/CAMAof68L0GO0F0bwuXtLZAjh9k_Hj%2Bo0-8mqfO6iEQyXr4PuVA%40mail.gmail.com), [#122](https://www.postgresql.org/message-id/CAEze2WhcnPrb1F2urn1M3RAm292FXkTJ5zXQ%3DdCUyab47JJ0mw%40mail.gmail.com), [#123](https://www.postgresql.org/message-id/CADzfLwW%3Dq71fw0hye9K3vBnS7RR50rTcCM91ZbWkiKeU%2B5s4_g%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. A stub index AM in pg_am that every AM-wide change must special-case

A future patch adds a new index AM callback or a new code path that loops over indexes, such as ANALYZE, VACUUM, pgstattuple or amcheck. It does not handle STIR's stub behaviour. A leftover `_ccaux` index then raises an error or an assertion, as the unclosed critical section did during ANALYZE in message #62, and users find out only in the field.

The series registers `stir` in pg_am with an opclass and opfamily, and special-cases it in analyze.c, vacuumlazy.c, vacuumparallel.c, pgstattuple and opr_sanity. Heikki in message #106 proposed a shared memory struct instead of using the catalog for this coordination. Message #107 answered with relcache-synchronization concerns, and the thread did not settle which design leaves fewer places to update. The material does not say what makes a future change fail loudly when it forgets STIR.

**Question:** What makes a new index-wide code path fail in regression tests, not in production, when it does not account for a STIR index?

<sub>fixable, level 2, catalog changes; messages [#62](https://www.postgresql.org/message-id/CAMAof69JSL8MYWG2qRScs3RQDpfcyZT_wFwW4SoAvftW%2BK_p1g%40mail.gmail.com), [#106](https://www.postgresql.org/message-id/4b368c17-a614-401b-a335-398450249c47%40iki.fi), [#107](https://www.postgresql.org/message-id/CADzfLwWUM2H2%2BdaMv9q0%2BroX_NGe3cPT%2BAgvQ60Vh_J-e3ts6Q%40mail.gmail.com), [#108](https://www.postgresql.org/message-id/CAEze2WjjL2Uee7ZZzFr%2BsDe9jdFtaoX6TWdry6NRvGr2FX0ASw%40mail.gmail.com); `maintenance-fit-2`</sub>

### 5. Docs promise minimal VACUUM impact before the series delivers it

A DBA reads the new CREATE INDEX and REINDEX text that concurrent builds have minimal impact on VACUUM of other tables. The DBA then runs a long CIC on a large table. The first heap scan still holds one snapshot, because the first-scan resets now live in CF entry 6401, so the xmin horizon stays behind for the whole build and other tables bloat.

The documentation hunk comes from the full series, but message #74 removed the first-phase snapshot reset from this entry. Message #125 says the two series work independently and still leaves a TODO for skipped TIDs. If this entry is committed alone, the user-facing contract claims more than the code delivers, and that wording is hard to take back once released.

**Question:** What exactly will the CREATE INDEX and REINDEX docs promise about VACUUM if this entry is committed without CF entry 6401?

<sub>fixable, level 1b, design; messages [#74](https://www.postgresql.org/message-id/CADzfLwVaV15R2rUNZmKqLKweiN3SnUBg%3D6_qGE_ERb7cdQUD8g%40mail.gmail.com), [#110](https://www.postgresql.org/message-id/CADzfLwVPQOF7m5x4d6%3D1tRJO%3D%3DFUZdc3LeO%3DN_TpmOzii7iH_Q%40mail.gmail.com), [#125](https://www.postgresql.org/message-id/CADzfLwUykLAjNNp0cMNMErMmb6%2BAJrDTiRuvvkA0dUOrzX3x1g%40mail.gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. Which invariant ensures that every live tuple missing from the target index has its HOT-root TID in STIR when pruning runs between validation snapshot refreshes?
2. Which CF entry owns the handoff of TIDs skipped by the build to validation, and which test fails if entry 6401 lands without it?
3. What do SET LOGGED, SET UNLOGGED, VACUUM FULL, CLUSTER and TRUNCATE do to a leftover unlogged `_ccaux` index on a permanent table?
4. What makes a new index-wide code path fail in regression tests when it does not account for a STIR index?
5. What will the CREATE INDEX and REINDEX documentation say about VACUUM if this entry is committed without CF entry 6401?

## Touch points

- **on-disk format / pg_upgrade:** A failed build leaves an unlogged `_ccaux` STIR index on disk, attached to a permanent table.
- **WAL / replication / standby:** The STIR index writes no WAL, but its catalog state is WAL-logged and replicated.
- **catalog changes:** The patch adds `stir` rows to pg_am, pg_opclass, pg_opfamily and pg_proc, and a new dependency link.
- **ABI / extension API:** `index_create()` gets new parameters; IndexInfo, IndexVacuumInfo, tableam.h and tuplestore change.
- **psql UX conventions:** psql `\d` shows a second INVALID index, `idx_ccaux` with AM `stir`, after a failed build.
- **maintenance burden ("who maintains this in 10 years"):** A new index AM is special-cased in ANALYZE, VACUUM, parallel vacuum and pgstattuple, and correctness is shared with CF entry 6401.
- **performance on the hot path:** Every insert into a table under a concurrent build also inserts a TID into the STIR index.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas supported the STIR approach in the thread, but v40 has no reply. The owner must know HOT pruning, snapshot and xmin handling, and the concurrent index build code.
- Priority: proceed; audience a sizeable minority; review cost high. Operators of large, busy tables gain faster concurrent builds and a moving xmin horizon, and several reviewers asked for this. The series is large, adds a catalog AM and touches heap, nbtree, catalog and vacuum code; it needs a rebase and a written safety argument.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 7.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
