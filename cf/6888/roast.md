# Design pre-review: Allow wal_log_hints to be changed without restart

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6888](https://commitfest.postgresql.org/patch/6888/); [thread](https://www.postgresql.org/message-id/flat/ajAAwSFy0WVMroyk%40bdtpg), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is understood and has reviewer support, but the design fails at level 2. Message #2 reproduces wrong tuple visibility after pg_rewind when wal_log_hints was off after the divergence and then reloaded to on (replication-ondisk-1), and the author has not replied. The proposal also does not define the value on standbys and at promotion (replication-ondisk-2). Demand is thin (demand-1, already-exists-1), but that alone would not stop review of a small patch.

Not checked: Without the code, crash safety during the change, the handling of the shared flag in recovery, and the cost of the shared read in XLogHintBitIsNeeded() could not be judged.

## Prior attempts

None found before 2026-06-15.

## Objections a reviewer is likely to raise

### 1. pg_rewind accepts a target that had wal_log_hints off after divergence

An operator turns wal_log_hints off with a reload, a failover happens, and the operator then reloads it back on. pg_rewind passes its pg_control check but misses hint-bit-only page changes made during the off period. The rewound node shows a row that does not exist on the source, and nobody finds out until queries return wrong results.

Message #2 reproduced this and noted that the patch's own 012_wal_log_hints.pl runs this exact sequence and reports success. The claim in message #1 that the current pg_control value is enough contradicts the old thread it cites, which asks for the value to be on continuously from the fork point. A restart can already create the gap, but a reload makes it a routine action. The author has not replied, and the patch adds no check on the pg_rewind side.

**Question:** How will pg_rewind refuse a target whose WAL after the fork point contains an XLOG_PARAMETER_CHANGE with wal_log_hints off, as message #2 proposes?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Effective value on standbys and after promotion is not defined

A standby has wal_log_hints = off in its own postgresql.conf and replays a primary that runs with it on. After promotion, the new primary's shared flag and pg_control may hold the local or the replayed value. A later pg_rewind of this node, or of its peer, then trusts a value that never matched the WAL it wrote.

The proposal describes only the checkpointer on a running primary. It does not say what replay of XLOG_PARAMETER_CHANGE does to XLogCtl->walLogHints or to pg_control on a standby. It also does not say when the local setting takes effect at promotion, or whether a record is written then. Today the setting is fixed at start, so these cases did not need an answer before this patch.

**Question:** What value does a standby hold during replay and at promotion, and which record shows a switch to the local setting in WAL that pg_rewind scans?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg); `replication-ondisk-2`</sub>

### 3. Demand rests on one measurement use case, with no field reports

The patch goes in for the few clusters still running without checksums whose owners want a WAL estimate. Since 04bec894a04 that group shrinks each release. Those owners already estimate today with one planned restart or on a staging clone, so the new code path serves a one-time task.

Message #1 gives a single motivation: avoid one restart before an online checksum change. It cites no -general or -bugs report and no user who asked for this. Message #2 adds a +1 but no use case of its own. With checksums on by default, the users left are those who chose otherwise or came through an upgrade, and the material does not size that group.

**Question:** Which users today keep checksums off, want to measure hint-bit WAL first, and cannot afford one restart or a test on a clone?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com); `demand-1`</sub>

### 4. Measuring checksum WAL cost already works without a production restart

An operator wants the WAL cost of checksums on a cluster that runs without them. Today they can enable data checksums online and watch steady-state WAL, then disable them online if the cost is too high, or restart a clone with wal_log_hints = on. The patch adds a reloadable path, and with it the pg_rewind gap shown in message #2, so that this one test step avoids a restart.

Message #1 gives the measurement as the main use, and the online checksum feature from f19c0ecca can already do that without a restart. Enabling checksums online likely costs a full pass that writes every page, so the existing way may be expensive on large clusters. That makes it less convenient, but it still works. A committer would likely ask whether that gap is worth a permanent shared flag plus a new ordering rule in the checkpointer.

**Question:** Which users cannot measure checksum WAL cost by enabling checksums online or by testing a restarted copy, and why is that cost too high for them?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. How will pg_rewind refuse a target whose WAL after the divergence point contains an XLOG_PARAMETER_CHANGE record with wal_log_hints = off, as message #2 proposes?
2. What value does XLogCtl->walLogHints hold on a standby during replay, after a standby reload, and at promotion, and which WAL record shows a switch?
3. What does pg_control report for wal_log_hints between the reload and the checkpointer's update, and can pg_rewind read a stale value after a crash in that window?
4. Which users keep data checksums off, want to measure hint-bit WAL first, and cannot use one planned restart or a restarted clone?
5. How does a reload of wal_log_hints behave while data checksums are being enabled or disabled online at the same time?

## Touch points

- **on-disk format / pg_upgrade:** The wal_log_hints value in pg_control can now change while the server runs, and pg_rewind reads it.
- **WAL / replication / standby:** The checkpointer writes XLOG_PARAMETER_CHANGE on reload, and pg_rewind correctness depends on the setting's history.
- **ABI / extension API:** The patch touches xlog.h and changes what XLogHintBitIsNeeded() reads.
- **new GUC:** No new GUC, but wal_log_hints changes context from PGC_POSTMASTER to PGC_SIGHUP.
- **backpatchability:** Message #2 says a restart can already create the same pg_rewind gap, so a pg_rewind check may concern released branches.
- **maintenance burden ("who maintains this in 10 years"):** A shared flag, an ordering rule in the checkpointer, and a likely new pg_rewind check on scanned WAL.
- **performance on the hot path:** XLogHintBitIsNeeded() now reads a shared-memory flag on the hint-bit path.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** One planned restart or a restarted clone already gives the WAL measurement.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows xlog.c parameter-change logging, the checkpointer, and pg_rewind's WAL scan; none has spoken in the visible thread.
- Priority: fix design first; audience niche; review cost medium. The patch is small and follows the full_page_writes pattern, but it serves a one-time measurement for clusters without checksums. The reproduced pg_rewind inconsistency in message #2 needs an agreed fix before code review is useful.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
