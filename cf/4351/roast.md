# Design pre-review: Index Prefetching

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4351](https://commitfest.postgresql.org/patch/4351/); [thread](https://www.postgresql.org/message-id/flat/cf85f46f-b02f-05b2-5248-5000b894ebab%40enterprisedb.com), 515 of 515 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the demand, and committers who know the area wrote the series. Four fixable design points remain at level 2. They are the GiST hot standby interlock (replication-ondisk-1), the crash path for table AMs without the batch callbacks (maintenance-fit-1), the lost SP-GiST index-only scans (standards-compat-1) and the changed index tuple counter (standards-compat-2). None of them requires a different proposal.

Not checked: The v38 code was not read. Whether RLS and leakproof quals are applied in the same order after batching, and how the full series performs across all index AMs and io_method settings, could not be checked.

## Prior attempts

None found before 2023-06-08.

## Objections a reviewer is likely to raise

### 1. GiST batches on a hot standby can return deleted tuples

On a hot standby, a cursor runs an index-only scan on a GiST index and holds a batch copied from the root leaf. The primary splits the root, then vacuums the heap and sets all-visible bits. Replay applies this while the cursor still holds the batch, so the cursor resumes, skips heap checks and returns rows already deleted on the primary.

Message #502 reports that the GiST cleanup-lock interlock is enforced only when VACUUM deletes leaf tuples, so the standby has no record that conflicts with the pinned root. Message #507 answers that nbtree has the same bug on master, which does not settle the point. Moving GiST to amgetbatch makes it hold copied items across pins for longer, the same pattern that exposes the bug. The thread summary lists this as an open point.

**Question:** Which WAL record makes standby replay wait for a GiST scan that still holds a batch from the old root, or why does amgetbatch not make this window wider?

<sub>fixable, level 2, WAL / replication / standby; messages [#502](https://www.postgresql.org/message-id/ECCA330B-75BB-43BF-96E9-9049BF213019%40gmail.com), [#507](https://www.postgresql.org/message-id/CAH2-WznVcopv-p42CqmJ3FzP3EXw41nWFCeEKi1jRo3csWRoZA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Out-of-core table AMs lose btree index scans without the batch callbacks

An extension table AM that works on Postgres 19 is loaded on Postgres 20. An assert build fails the new GetTableAmRoutine() assertion at load. A release build calls a NULL amgettuple on the first btree scan and crashes.

The brief lists index_scan_batch_init, index_scan_markpos and index_scan_restrpos as new required callbacks, and nbtree sets amgettuple to NULL. Message #515 points out this break, and the thread has no answer yet. The proposal does not say how tableam.sgml documents it, and it has no generic fallback that simple table AMs could use. Each table AM author would then copy the ring-buffer logic, which is the duplication Andres warned about in message #484.

**Question:** Will a generic batch implementation in tableam_indexscan.c let an existing table AM keep working, and where is the break documented?

<sub>fixable, level 2, ABI / extension API; messages [#515](https://www.postgresql.org/message-id/auOon54DuG2A6ll4%40floki), [#484](https://www.postgresql.org/message-id/6sphk3ycctmbihlrykts7uj6mjakop6wrq2dhe3vnlmrnldz2f%40uuwmkd6jjrxa); `maintenance-fit-1`</sub>

### 3. SP-GiST long-value opclasses lose index-only scans after upgrade

A site uses an SP-GiST text radix index for index-only prefix lookups on a mostly all-visible table. After upgrade the planner can no longer pick an index-only scan, so every match fetches a heap page. The site finds out when latency rises or EXPLAIN stops showing Index Only Scan, with no warning at upgrade.

Message #491 states this is "a legitimate limitation in the amgetbatch design itself", with no existing bug behind it, and proposes to accept it so SP-GiST can prefetch. That withdraws a working plan type from existing indexes in exchange for a gain on another plan type. The GiST ordered index-only scan removal in message #489 is a bug fix and differs. The SP-GiST loss has no stated user notice: Tomas's documentation patch in message #509 covers only nearest-neighbor ordering.

**Question:** Which users run index-only scans on SP-GiST long-value opclasses today, and why is keeping amgettuple for those opclasses not viable?

<sub>fixable, level 2, planner behaviour change; messages [#491](https://www.postgresql.org/message-id/CAH2-WzkZTkDuyVFszLwPJesF9pS5E8m0UA%2B344bx-B-zfA5kaw%40mail.gmail.com), [#489](https://www.postgresql.org/message-id/CAH2-WzmAKApAYttMQfhh-usweXvRYpHcku-OKHYgDvb%3DwLSWHg%40mail.gmail.com), [#509](https://www.postgresql.org/message-id/fcf9030d-150c-4ce8-89f5-83c8e5f4355f%40vondra.me); `standards-compat-1`</sub>

### 4. pg_stat index tuple counters change meaning without user notice

A monitoring tool alerts on the idx_tup_read to idx_tup_fetch ratio in pg_stat_user_indexes. After upgrade, ORDER BY ... LIMIT 1 queries count a whole leaf page of index tuples per call, so the ratio jumps and the alerts fire. The DBA cannot tell a counting change from index bloat.

Message #492 moves pgstat_count_index_tuples to once per batch for a micro-optimization, and message #494 says the change could be "several hundred times more" and "isn't a tiny behavioral change". The argument for it is consistency with bitmap scans, but users have read this counter per plain index scan for many releases. The proposal does not say where monitoring.sgml or the release notes would describe the new meaning.

**Question:** Could the counter keep its per-tuple meaning at about the same cost, and if not, where is the change documented for monitoring users?

<sub>fixable, level 2, design; messages [#492](https://www.postgresql.org/message-id/CAH2-WzmsebafEMoBp-sWq%2BDp75SKV6j%2BRWerKSJNfJkWV592tg%40mail.gmail.com), [#493](https://www.postgresql.org/message-id/0352479d-6a59-4e80-8ad1-12053a4ad148%40vondra.me), [#494](https://www.postgresql.org/message-id/CAH2-Wz%3DDhG5%3Dsw9P1A5%3DbnXxCz4UJaWgH5WQA-3pEckXRucZdQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which WAL record makes standby replay wait for a GiST scan that still holds a batch from the old root page?
2. Will a table AM without the batch callbacks get a generic batch implementation or a clean load-time error, and where does tableam.sgml document the change?
3. Why can SP-GiST long-value opclasses not keep amgettuple, so that their index-only scans stay available?
4. Can pgstat_count_index_tuples keep its per-tuple meaning at about the same cost, and if not, where will monitoring users read about the change?
5. What are the benchmark results for the full v38 series across nbtree, hash, GiST and SP-GiST with each io_method setting?

## Touch points

- **WAL / replication / standby:** GiST scans on a hot standby hold batches while replay sets all-visible bits (messages #502, #507).
- **ABI / extension API:** New amgetbatch, amkillitemsbatch and amgettransform index AM callbacks, and three new required table AM callbacks.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: SP-GiST long-value opclasses lose index-only scans after upgrade
- **new GUC:** The series adds the developer option debug_disable_indexscan_prefetch.
- **planner behaviour change:** The planner can no longer pick index-only scans for GiST and SP-GiST ordering operators or SP-GiST long-value opclasses.
- **backpatchability:** The thread asks whether the SP-GiST INCLUDE fix should be back-patched.
- **maintenance burden ("who maintains this in 10 years"):** Four index AMs and every table AM must follow the new batch interface and the read stream changes.
- **performance on the hot path:** Every plain index scan and index-only scan goes through the batch path; a start heuristic guards LIMIT and nestloop cases.

## Who would commit it, and when

- Owner: qualified committer engaged. The CommitFest entry lists Peter Geoghegan and Tomas Vondra as authors. The owner needs to know nbtree, the index AM and table AM APIs, and the read stream code.
- Priority: proceed; audience most users; review cost high. Index scans that read uncached heap pages are common, and message #1 reports about 16 s down to 1.8 s for 100000 rows. The series changes the index AM and table AM APIs and four index AMs, so review and upkeep are costly, but qualified committers are engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
