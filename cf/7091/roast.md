# Design pre-review: Redesign per-backend statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7091](https://commitfest.postgresql.org/patch/7091/); [thread](https://www.postgresql.org/message-id/flat/anCjoFkGah%2FuB7lw%40bdtpg), 15 of 15 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b, and two committers who know pgstat asked for the fixed array. Two fixable level 2 points remain. maintenance-fit-1 asks for query-latency numbers for the array version, because the thread only measured the dshash version. standards-compat-1 asks what a shared reset does to each live entry's stats_reset.

Not checked: Without the code, the locking on unused slots and the transfer at exit, ProcNumber reuse and shutdown could not be checked for lost or double-counted counters. Only 3 of 15 messages were shown, so some open questions may already have answers in the thread.

## Prior attempts

- [Get rid of pgstat_count_backend_io_op*() functions](https://www.postgresql.org/message-id/flat/aLWonxez0MBSqEvA@ip-10-97-1-34.eu-west-3.compute.internal) (similar idea; Rejected, 2026-03-25). "Hmm. I really have an issue with the ordering dependency this introduces between stats kinds, assumed to happen inside the code." ([message](https://www.postgresql.org/message-id/aNTkWRZAg9FCBKy0%40paquier.xyz)) Different now: Both remove the double counting of per-backend I/O, but P1 copied pending I/O stats into the backend kind and needed a flush order between kinds. Here each kind owns its own ProcNumber-indexed entries, so no copy and no flush ordering remain.

## Objections a reviewer is likely to raise

### 1. Global view query cost scales with MaxBackends, unmeasured for the array design

A site with max_connections = 5000 runs a monitoring agent that polls pg_stat_io every few seconds on each node. Every query now takes one shared LWLock per array slot and adds up every entry. The operator sees higher monitoring latency, and more time spent in LWLock calls, only after upgrading.

The only latency numbers are for the dshash version (message #5), where pg_stat_io went from about 0.07 ms to 2.8 ms at 10000 backends. Message #10 says a fixed array must also visit unused slots, and the v4/v5 array design has no numbers at all. pg_stat_io, pg_stat_wal and pg_stat_lock are read by nearly every monitoring tool, so the cost falls on everyone, including people who never call pg_stat_get_backend_*(). Each later per-backend kind adds another scan of this kind.

**Question:** What do pg_stat_io, pg_stat_wal and pg_stat_lock cost with v5 at a few common max_connections values, with most slots idle or unused, compared with master?

<sub>fixable, level 2, performance on the hot path; messages [#4](https://www.postgresql.org/message-id/anlVx_TD_Q8He5uf%40paquier.xyz), [#5](https://www.postgresql.org/message-id/annj%2FETYOFgbrGsn%40bdtpg), [#10](https://www.postgresql.org/message-id/arHw6cycX%2Bpe53wX%40bdtpg), [#12](https://www.postgresql.org/message-id/at8%2FezwmjHCWRmh2%40bdtpg), [#13](https://www.postgresql.org/message-id/auwR7e5mba17AISr%40paquier.xyz); `maintenance-fit-1`</sub>

### 2. Shared reset now clears live per-backend counters without a stated contract

A monitoring agent samples pg_stat_get_backend_io() for each pid and graphs deltas. A DBA runs pg_stat_reset_shared('io'), and every live backend's counters drop to zero. Unless the per-backend stats_reset timestamp also changes, the agent reports negative or wrapped deltas and finds out only from wrong graphs after upgrade.

On master, a shared reset leaves per-backend counters alone, and pg_stat_reset_backend_stats() resets them. The doc hunks in 0003-0005 add the new reach of pg_stat_reset_shared() but do not say whether each live entry's stats_reset is updated. Message #1 calls this "a documented behavior change", but the thread has no release-note text and no rule for how clients tell a shared reset from a per-backend one. Tools that already use the per-backend functions upgrade into this silently.

**Question:** After pg_stat_reset_shared('io'), does each live backend's per-backend stats_reset change, and what release-note text tells tool authors about the new reset reach?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/anCjoFkGah%2FuB7lw%40bdtpg); `standards-compat-1`</sub>

## Questions for the author

1. What do pg_stat_io, pg_stat_wal and pg_stat_lock cost with v5 at a few common max_connections values, with most slots unused, compared with master?
2. After pg_stat_reset_shared('io'), does each live backend's per-backend stats_reset change, and where do the docs say so?
3. Is there a flush-path benchmark against master that shows the reduced lock contention claimed in message #1?
4. How much shared memory do the three arrays use at common max_connections values, and does the size grow with max_worker_processes and other process slots?
5. Does a global view query take the LWLock of unused array slots, or does it skip them without locking?

## Touch points

- **WAL / replication / standby:** 0005 touches walsender.c, and per-backend rows now cover the startup process and WAL-related auxiliary processes.
- **ABI / extension API:** PgStat_KindInfo gains per-backend metadata, PGSTAT_KIND_BACKEND is removed, and custom kinds with per-backend entries fail to register.
- **backpatchability:** see objection standards-compat-1: Shared reset now clears live per-backend counters without a stated contract
- **maintenance burden ("who maintains this in 10 years"):** New generic per-backend infrastructure with per-entry LWLocks and transfer at exit, ProcNumber reuse and clean shutdown.
- **performance on the hot path:** Flushes change locking, and pg_stat_wal, pg_stat_lock and pg_stat_io now add up every array slot.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier reviews the series in the visible thread, and the commit messages list him and Andres Freund as reviewers; both asked for the fixed array.
- Priority: proceed; audience a sizeable minority; review cost medium. The series removes duplicate accounting that reviewers raised and unblocks further per-backend statistics, and committers who know pgstat are engaged. Six staged patches stay within the pgstat code, and the two open points need numbers and one documented answer.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 3.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
