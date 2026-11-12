# Design pre-review: improving handling of FPIs during recovery prefetching

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7512](https://commitfest.postgresql.org/patch/7512/); [thread](https://www.postgresql.org/message-id/flat/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the basic design. Three fixable level 2 points remain: maintenance-fit-2 (no measurement of the lookup cost for replay that gains nothing), replication-ondisk-1 (the squashed commit ties the fix to a catalog change, so it cannot be back-patched as is), and standards-compat-2 (the skip_fpi name). Message #1 shows demand from a field report and an earlier patch. Confidence is medium because the code was not read and a simpler alternative is expected but not posted.

Not checked: Without the code, the correctness of the eviction by replaying_lsn, the reset paths and the group overflow behaviour could not be checked. The expected simpler alternative is not posted, so no comparison was possible.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. Lookup cost on every block reference is unmeasured

A standby that replays mostly non-FPI records, with prefetch already working well, now does a hash and a group scan for each block reference whenever any FPI is pending. If that cost adds up, replay gets slower on standbys that never had the IOPS problem. The project finds out from user reports after release.

In message #6 the author asks others to check the CPU cost for normal cases and says he could not build a bad case himself. Jan's numbers in message #3 only count posix_fadvise() calls on the workload that gains from the patch. Patch 0009 exits early only when no FPIs are pending, which is rare right after a checkpoint, and the history can grow to about 400kB at maintenance_io_concurrency=1000.

**Question:** How much does replay throughput change on a CPU-bound standby with few useful skips, at default and high maintenance_io_concurrency?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/e1b36cd3-c3a5-4da0-a1d7-497296d5d9f3%40planetscale.com), [#6](https://www.postgresql.org/message-id/60a1221b-125b-481c-b019-66366a9e2673%40vondra.me); `maintenance-fit-2`</sub>

### 2. Stats column ties an outage fix to master only

An operator on Postgres 15 to 19 runs a replica under an IOPS limit and keeps seeing replay lag after a walreceiver restart. The squashed commit changes pg_proc.dat and system_views.sql, which needs a catversion bump, so it cannot go to back branches as one commit. The operator finds out only when the fix ships in a future major release.

Message #1 links recovery_prefetch to outages, but the only stated way out on released branches is recovery_prefetch = off. Patch 0002 mixes the behaviour change with the skip_fpi column in pg_stat_recovery_prefetch and a changed function in pg_proc.dat, and the plan squashes all nine patches into one commit. A committer would likely ask for the in-memory FPI history to be split from the catalog-visible counter, so the history alone can be backpatched.

**Question:** Is the FPI history meant for back branches, and if so, can it ship without the skip_fpi column and the pg_proc.dat change?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me); `replication-ondisk-1`</sub>

### 3. skip_fpi sits next to skip_fpw with a confusingly similar meaning

A DBA reading pg_stat_recovery_prefetch sees skip_fpw and skip_fpi side by side and sums them, or graphs the wrong one. After release, monitoring tools hard-code the column name, and renaming it later breaks their queries.

The 0002 commit message says the name "is not very self-explanatory" and "easy to confuse with skip_fpw", and nothing in the thread picks a better one. Both columns count blocks skipped because of a full page image, one for the record that carries it and one for later records. View column names become part of the monitoring contract once released, so a committer would likely ask for the name and the doc text to be settled before commit.

**Question:** What column name and documentation wording make the difference from skip_fpw clear to someone who reads only the view?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/b39398cc-fa51-45c6-b467-57537e4b5da1%40vondra.me); `standards-compat-2`</sub>

## Questions for the author

1. How much does replay throughput change on a CPU-bound standby with few useful skips, at default and at high maintenance_io_concurrency?
2. Is the FPI history meant for back branches, and if so, can it ship without the skip_fpi column and the pg_proc.dat change?
3. What column name and documentation wording make the difference from skip_fpw clear to someone who reads only the view?
4. Should the history also track BKPBLOCK_WILL_INIT blocks, as message #2 suggests, since redo reads no page in that case either?
5. Will problem (2), the slow restart of streaming after a walreceiver exit, be handled in a separate thread?

## Touch points

- **WAL / replication / standby:** The patch changes which blocks the WAL prefetcher reads ahead during recovery on standbys and in crash recovery.
- **catalog changes:** 0002 changes pg_proc.dat and system_views.sql to add skip_fpi to pg_stat_recovery_prefetch.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: skip_fpi sits next to skip_fpw with a confusingly similar meaning
- **backpatchability:** Message #1 links the behaviour to outages on released branches, and the squashed commit includes a catalog change.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a second cache in xlogprefetcher.c with its own sizing and reset rules, plus a new view column.
- **performance on the hot path:** The prefetcher does a hash and a group scan for each block reference while any FPI is pending.

## Who would commit it, and when

- Owner: qualified committer engaged. The author, Tomas Vondra, says in message #1 that he was involved in getting recovery_prefetch into PG 15. The owner must know the WAL prefetcher and the redo read path.
- Priority: proceed; audience a sizeable minority; review cost medium. Replicas under IOPS limits gain, and message #3 shows posix_fadvise() calls dropping from 23,502 to 821. The change is mostly in xlogprefetcher.c plus one view column; review cost is set to medium because no code was read and a simpler alternative may replace the series.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
