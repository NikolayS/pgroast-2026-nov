# Design pre-review: Expose checkpoint timestamp and duration in pg_stat_checkpointer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6307](https://commitfest.postgresql.org/patch/6307/); [thread](https://www.postgresql.org/message-id/flat/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), 15 of 15 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The review stops at level 1b. The thread cites no monitoring tool or operator that asked for last-only gauges (demand-1). pg_control_checkpoint(), the num_* counters and the total= field of the log line already give the same data (already-exists-1). The "checkpoint starting:" line already prints the checkpoint flags (already-exists-2). The log change has support in message #2, so the goal may be fine once the author answers these points.

Not checked: No full, split patch set was available. Without the code, this review could not check how reset, flush and restartpoints treat the gauges, or which flags the complete line prints.

## Prior attempts

- [Add last_vacuum_index_scans in pg_stat_all_tables](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com) (similar idea; Withdrawn, 2023-01-31). "No other stats counter works that way; they are all additive." ([message](https://www.postgresql.org/message-id/20220708164052.bp62uhma56v7u7n5%40alvherre.pgsql)) Different now: Same pattern of a last-run-only value in a cumulative stats view, applied to checkpoint duration instead of vacuum index scans; the additive-stats objection is not answered.
- [add the last checkpoint kind to the control file](https://www.postgresql.org/message-id/flat/CALj2ACUSED8QQWDHZ6yPusH7p7VecQ0pNm5Oin7WwdxkyhhUWg@mail.gmail.com) (related; Withdrawn, 2022-02-03). "For now we have some room, but we will likely keep consuming bytes in that file for more critical features and it's almost certain that at one point we will regret wasting 2 bytes for that." ([message](https://www.postgresql.org/message-id/20220131055416.iq7yknigyuxcm6tj%40jrouhaud)) Different now: Reports the last checkpoint kind in the completion log line instead of the control file, so the control-file space concern does not apply; the upgraded-flags issue may still apply.

## Objections a reviewer is likely to raise

### 1. No user demand shown for last-only checkpoint gauges

A monitoring vendor polls pg_stat_checkpointer every minute while checkpoints run every 20 seconds during a write spike. Its alert sees one duration out of three and misses the slow one. Everyone then keeps two new columns and functions that the target users cannot rely on.

The only use case offered is the author's own debugging session (message #1), plus a hypothetical agent in message #14. No field report, -general thread or monitoring tool asking for this is cited. Michael Banck stayed unconvinced (messages #2, #11, #13), Álvaro Herrera wanted a sleep-time breakdown first (message #3), and the May 2026 revive (message #15) got no reply. [P4] was withdrawn after a similar last-only column met the objection that stats counters are additive.

Precedent: [Add last_vacuum_index_scans in pg_stat_all_tables](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com) (Withdrawn, 2023-01-31)

**Question:** Which monitoring tool or operator has asked for a last-checkpoint duration, and why do cumulative counters plus log_checkpoints not meet that need?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/20251124091822.GC8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#3](https://www.postgresql.org/message-id/202511240955.vt3fjrb4ksrs%40alvherre.pgsql), [#11](https://www.postgresql.org/message-id/20251128080849.GA13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#13](https://www.postgresql.org/message-id/20251201081548.GE13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#14](https://www.postgresql.org/message-id/CAMtXxw_JK-YbpM9fgqnv9eAFEKoGx38Aq_tNFs0jCZL4C_4nyg%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAMtXxw9n%3DgOKYti2ekvPJ7tS7MQewgrkSmO8do-A%2BKwXXdoLLw%40mail.gmail.com); `demand-1`</sub>

### 2. Per-checkpoint duration and time are already in logs and pg_control_checkpoint()

A monitoring tool reads last_checkpoint_time from pg_stat_checkpointer and checkpoint_time from pg_control_checkpoint(). The first is the checkpoint end and the second is its start, so they never match. Users find out when dashboards disagree, and they have two "last checkpoint" sources to keep apart.

pg_control_checkpoint() already returns the time of the latest checkpoint or restartpoint. The "checkpoint complete" line, logged by default under log_checkpoints, already gives total= per checkpoint, which pgbadger reads (message #7, #8). A poller can detect frequency from num_timed, num_requested and num_done. The author says this is "hard to get" without delta math (message #14), but that describes less convenient, not impossible.

**Question:** What can a tool do with the two new columns that it cannot do with pg_control_checkpoint(), the num_* counters and the log_checkpoints total= field?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#7](https://www.postgresql.org/message-id/CAC%2BAXB1Vco9jSmjZsdGupqiRMSDfTBu_H6r%3DFQhmZrUpwScjhA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/20251126174138.GE8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#13](https://www.postgresql.org/message-id/20251201081548.GE13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#14](https://www.postgresql.org/message-id/CAMtXxw_JK-YbpM9fgqnv9eAFEKoGx38Aq_tNFs0jCZL4C_4nyg%40mail.gmail.com); `already-exists-1`</sub>

### 3. Checkpoint type is already logged in the checkpoint starting line

With log_checkpoints on, each checkpoint already emits "checkpoint starting: immediate force wait" or "checkpoint starting: wal". The patch repeats a subset of those flags on the complete line. Log parsers that match the complete line must change, and the two lines can show different flag sets.

Michael Banck and the author both treat the type as missing from the log (message #2, #5), but the starting line lists the flags. Pairing a start line with the next complete line is a documented, modest step that pgbadger already performs. The proposal names only manual, timed and immediate, which is fewer than the starting line shows, such as wal and end-of-recovery.

**Question:** What does the type on the complete line give that the existing checkpoint starting line, which already lists the flags, does not?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/20251124091822.GC8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#5](https://www.postgresql.org/message-id/CAMtXxw8%2BW0UCCXS71_u3TiFnO_f47aRQPq_ar8EJspapgAs5SQ%40mail.gmail.com); `already-exists-2`</sub>

## Questions for the author

1. Which monitoring tool or operator has asked for a last-checkpoint duration, and why do cumulative counters plus log_checkpoints not meet that need?
2. What can a tool do with the two new columns that it cannot do with pg_control_checkpoint(), the num_* counters and the log_checkpoints total= field?
3. What does the type on the complete line give that the existing "checkpoint starting:" line, which already lists the flags, does not?
4. Why is a last-only gauge better than an additive form, such as a cumulative total duration that a poller can turn into deltas?
5. On a standby, after promotion and after pg_stat_reset_shared('checkpointer'), what do last_checkpoint_duration and last_checkpoint_time show?

## Touch points

- **WAL / replication / standby:** The gauges are set where restartpoints also end, so standbys are affected and the proposal does not say how.
- **catalog changes:** The patch adds functions to pg_proc.dat and columns to pg_stat_checkpointer in system_views.sql.
- **ABI / extension API:** The patch adds fields to the PgStat_CheckpointerStats struct in pgstat.h.
- **maintenance burden ("who maintains this in 10 years"):** Overwritten fields sit in a struct of summed counters, and the changed log line reaches external log parsers.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** pg_control_checkpoint(), the num_* counters and the log_checkpoints lines already expose the last checkpoint time, duration and flags.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the cumulative statistics code and the checkpointer. Álvaro Herrera and Michael Banck commented early, but nobody replied to the May 2026 revive.
- Not now, show demand first: A request from a monitoring tool or operators for a last-checkpoint duration would change it. So would a concrete checkpoint problem that counter deltas, pg_control_checkpoint() and the log_checkpoints lines cannot detect.
- Priority: needs design rethink; audience niche; review cost low. The patch is about 7 KiB and touches xlog.c and the checkpointer stats code, but the gauges have no shown demand and need their own reset and flush rules. The log change has support and could go ahead alone once it is split out and reuses the flag words of the starting line.

---

[roast-full.md](roast-full.md) has 5 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
