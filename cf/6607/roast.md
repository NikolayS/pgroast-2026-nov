# Design pre-review: New pg_stat_tablespace view for tablespace level metrics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6607](https://commitfest.postgresql.org/patch/6607/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqRHsZw3%2BaeNZgnBYduQSY7qg1O4MBdmyFjK-A%2BTU1-b-A%40mail.gmail.com), 26 of 26 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Block time per tablespace has no existing source, so the goal can stand, but two design questions from Andres Freund are open after v12. The thread shows no benchmark for the new code in the buffer manager I/O path (maintenance-fit-2). The block and tuple counts repeat per-relation counters without an accepted reason (already-exists-1). Field demand rests on one synthetic demo (demand-1), which alone would only mean "show demand".

Not checked: The cost of the cached slot in SMgrRelation, the standby drop path and the grants on the new functions could not be checked without the code. The v12 column list and reset behaviour are cut off in the shown documentation.

## Prior attempts

None found before 2026-03-23.

## Objections a reviewer is likely to raise

### 1. New code in the buffer manager I/O path with no overhead numbers

Every installation runs the new path in bufmgr.c, localbuf.c and smgr.c, including those that never create a tablespace. A small regression in read-heavy workloads with track_io_timing on, or a wider SMgrRelation, would show up only after release. Other users of the feature would bear none of that cost alone.

Andres objected twice to duplicate counting in these paths (messages #16 and #23), and in message #23 he named cost details as small as argument passing on x86-64. v11 adds pgstat_count_smgr_io_op_time() and a cached slot in SMgrRelation, and the author describes the generated code in message #24. No benchmark is shown with track_io_timing on or off, and Andres has not said whether v11 or v12 meets his objection.

**Question:** What do read- and write-heavy pgbench or sequential-scan runs show for master versus v12, with track_io_timing on and off?

<sub>fixable, level 2, performance on the hot path; messages [#16](https://www.postgresql.org/message-id/qihwj3bbzazlcq4uxy3nfdnsekbdbbccsccmqyhv2q37oyl4t4%40f4jx4inuv2az), [#17](https://www.postgresql.org/message-id/CAGRkXqTTQ6pzTs1fKdUE%2BpzUHWryF5OQVVS9oZWUMV18wxa-fw%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/43fmu6aqpuvjpgy23odpwb3zheufz2vkbriuoda5wqweqcs4jm%40s2jo5e6xrlxl), [#24](https://www.postgresql.org/message-id/CAGRkXqQ%2B06oP4kKvhtVh3-QRZ4nDqcVqWb%3DTme6FnF5DoNyL_A%40mail.gmail.com); `maintenance-fit-2`</sub>

### 2. Entry creation spread over several code paths that are easy to miss

A future feature does relation I/O, writes a temp file, or changes reltablespace through a path that does not hit one of the creation points. Its counts are dropped without error, because a flush only looks up the entry. A DBA sees a tablespace with missing I/O and has no way to know.

The entry is now created at CREATE TABLESPACE, at the first timed I/O per SMgrRelation, on temp file use, and on a relation move (messages #22 and #26). Each of v6, v7, v10 and v12 fixed a lifecycle bug: a shutdown crash, a stats leak, and a lost entry after a crash and a move. Every reported case was fixed, but each fix added one more creation point, so the invariant is spread across files rather than held in one place.

**Question:** What single rule guarantees an entry exists before any count is flushed, and which assertion would catch a new path that breaks it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#11](https://www.postgresql.org/message-id/CAGRkXqSN5JZPTMou4KwXpq6AFOD2McWJBzjtsPjqt-2TiHw_2A%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAGRkXqQM90%3D58C%3Dht%2BnEFk7MTp1GMV8F3fw4Jg5z%2BNM3Wbu%3Dtg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAN4CZFOPiD-V8k5Pv7MrM-BzCK531p2HohA4%2BV5hNg8A73ysog%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CAGRkXqQsLfecnoA%2BNZvMW%2BQ2Z%2BnKGsdexeignD6utE0RWsHdQA%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAN4CZFMS8aMMu%3DEfdwPpnw8yrZGzA%3D%3Dqwo2tCru_BcLZQw-DsA%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAGRkXqQAP%2BjspenndKS-QPZ3RMBnJ4Kc_bLBdHCg8xROA%3DandA%40mail.gmail.com); `maintenance-fit-3`</sub>

### 3. Standby lifecycle of tablespace stats entries is not defined

On the primary, a DBA drops a tablespace and later creates a new one that reuses the OID. A standby replays both, but nothing removes or resets its stats entry, so the standby's pg_stat_tablespace shows the old counts under the new name. Nobody notices until standby numbers disagree with the primary after failover.

Message #11 resets stats in CreateTableSpace() and message #22 creates entries on first use, which covers standbys. Neither path runs during WAL replay, so the standby never reaches that reset. Relation and function stats avoid this by recording drops in the commit record so the standby replays them, and the proposal does not say whether tablespace drops use that path. Message #9 already fixed one standby crash, so the standby path is known to be fragile.

**Question:** Are tablespace stats drops recorded in the commit record and replayed on standbys, and what clears a standby entry when the primary reuses a tablespace OID?

<sub>fixable, level 2, WAL / replication / standby; messages [#9](https://www.postgresql.org/message-id/CAGRkXqRV02%3DiEhpOa1%3DJ8gtqwG5ZWs_TNDsqjPVNXggPW%3D8Mgg%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAGRkXqSN5JZPTMou4KwXpq6AFOD2McWJBzjtsPjqt-2TiHw_2A%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CAGRkXqQsLfecnoA%2BNZvMW%2BQ2Z%2BnKGsdexeignD6utE0RWsHdQA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Per-tablespace block and tuple counts repeat counters that already exist

A DBA can already sum pg_statio_all_tables and pg_stat_all_tables by pg_class.reltablespace in each database, with a monitoring tool connected to each one. With the patch, every relation stats flush also updates a second shared entry. Every future counter added to relation stats then needs a decision about the tablespace copy, and a missed one only shows up as a mismatch between views.

In message #23 Andres Freund asked why these counters are kept per tablespace when the querying side could compute them. Message #24 answers that relation stats stay inside one database and vanish when the relation is dropped. Both points make the query less convenient, but they do not make it impossible: tools already connect to every database, and periodic sampling keeps the deltas of dropped relations. The new data with no existing source is block time per tablespace, and that could be shipped without the duplicated counts.

**Question:** Which question about blks_hit, blks_read or tup_* per tablespace cannot be answered by summing per-relation stats across databases, sampled at monitoring intervals?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#23](https://www.postgresql.org/message-id/43fmu6aqpuvjpgy23odpwb3zheufz2vkbriuoda5wqweqcs4jm%40s2jo5e6xrlxl), [#24](https://www.postgresql.org/message-id/CAGRkXqQ%2B06oP4kKvhtVh3-QRZ4nDqcVqWb%3DTme6FnF5DoNyL_A%40mail.gmail.com); `already-exists-1`</sub>

### 5. No field demand shown for per-tablespace I/O timing

Every cluster pays for a new stats kind, its flush and a slot in the SMgrRelation I/O path. The users who gain run tablespaces on separate devices and today read per-device latency from iostat or /proc/diskstats. If that group is small, the cost on bufmgr.c lands on everyone and few people use the view.

The only use case is the author's own (message #1, #17), shown with a RAM disk next to an SSD under debug_io_direct=data (message #18). That is a demo, not a production report. Bernd's comment in message #12 that it "could be very useful" is a hope, not a user problem. Andres opens message #16 with "*If* we want this". Message #17 admits the OS tools answer the question, though less conveniently, and mapping a tablespace to its device is a one-time step for the DBA.

**Question:** Which users with tablespaces on separate devices have asked for this, and why do per-device OS statistics not answer their question in practice?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAGRkXqRHsZw3%2BaeNZgnBYduQSY7qg1O4MBdmyFjK-A%2BTU1-b-A%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/7f12ade9-1deb-4093-85c6-7c18271657ac%40gmx.at), [#16](https://www.postgresql.org/message-id/qihwj3bbzazlcq4uxy3nfdnsekbdbbccsccmqyhv2q37oyl4t4%40f4jx4inuv2az), [#17](https://www.postgresql.org/message-id/CAGRkXqTTQ6pzTs1fKdUE%2BpzUHWryF5OQVVS9oZWUMV18wxa-fw%40mail.gmail.com), [#18](https://www.postgresql.org/message-id/CAGRkXqQM90%3D58C%3Dht%2BnEFk7MTp1GMV8F3fw4Jg5z%2BNM3Wbu%3Dtg%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What do read-heavy and write-heavy runs show for master versus v12, with track_io_timing on and off?
2. Which question about blks_read, blks_hit or tup_* per tablespace cannot be answered by summing per-relation stats across databases at monitoring intervals?
3. What single rule guarantees a tablespace stats entry exists before any count is flushed, and which assertion catches a new path that breaks it?
4. Are tablespace stats drops replayed on standbys, and what clears a standby entry when the primary reuses a tablespace OID?
5. Which functions reset one tablespace and all tablespaces, and does the v12 view keep a stats_reset column?

## Touch points

- **on-disk format / pg_upgrade:** The stats file gains a new kind and the 2PC stats record gains a tablespace field; neither needs pg_upgrade work.
- **WAL / replication / standby:** Tablespace stats entries must be created, dropped and reset on standbys and after crash recovery.
- **catalog changes:** New system view pg_stat_tablespace, new pg_proc entries and a catversion bump.
- **ABI / extension API:** SMgrRelation gains a cached slot, and pgstat.h and pgstat_kind.h gain a new stats kind.
- **security / privileges:** The proposal does not say who may read the view or run the new functions and any reset.
- **maintenance burden ("who maintains this in 10 years"):** A third copy of relation counters, four entry creation points, and a new stats kind to keep in sync.
- **performance on the hot path:** pgstat_count_smgr_io_op_time() runs in bufmgr.c, localbuf.c and smgr.c for every cluster.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Per-device OS statistics and sums of per-relation stats already answer part of the question.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund reviewed the hot-path design in messages #16 and #23 but has not answered v11 or v12. The owner must know the buffer manager, smgr and cumulative stats code.
- Priority: fix design first; audience niche; review cost high. The view helps clusters with tablespaces on separate devices, and the 87.5 KiB patch touches bufmgr.c, smgr.c, relcache.c and the stats core. Benchmarks and a decision on the copied counters should come before more code review.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
