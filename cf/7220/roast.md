# Design pre-review: pg_rewind: Remove recovery at the start of rewind

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7220](https://commitfest.postgresql.org/patch/7220/); [thread](https://www.postgresql.org/message-id/flat/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The goal is reasonable and comes with a measurement, but the proposal removes a safety step without saying what replaces its side effects. replication-ondisk-2 is decisive: crash recovery used to bring pg_control up to date before pg_rewind read it, and message #1 names no replacement. maintenance-fit-1 and replication-ondisk-3 raise the same kind of gap for end-of-recovery actions and for the postmaster.pid interlock. All are answerable, and only one message is visible, so confidence is low.

Not checked: The code was not read. The patch may already derive the target timeline from WAL or check shared memory before it removes postmaster.pid; the prose does not say either.

## Prior attempts

- [Fix pg_rewind false positives caused by shutdown-only WAL](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com) (related; Withdrawn, 2026-05-01). "But I do not think we can do that, because (1) those WAL records might do important things like update the control file and (2) the server will not be OK with ignoring those WAL records even if pg_rewind decides that they are not important." ([message](https://www.postgresql.org/message-id/CA%2BTgmoYYvYShoRKK6%3D8PX-BYBarFXjrGB%2BcBBRjX-SkMfd0Nrg%40mail.gmail.com)) Different now: P8 skipped shutdown-only records when finding the end of target WAL. This proposal reads every record but no longer runs the recovery that updated the control file, so only the control-file part of the P8 objection carries over.

## Objections a reviewer is likely to raise

### 1. Target timeline and control-file fields are read without recovery updating them

A node is promoted, writes on the new timeline, and crashes before its first checkpoint after promotion completes. Its pg_control still shows the old timeline in checkPointCopy. pg_rewind builds the target history from that stale timeline, picks the wrong divergence point or reports that no rewind is needed, and the node later follows the new primary with diverged blocks.

Crash recovery used to end with a shutdown checkpoint that brought pg_control's checkpoint, timeline and minRecoveryPoint up to date before pg_rewind read them. [P8] was withdrawn partly because WAL after divergence updates the control file, and pg_rewind's sync decision depends on that. The proposal does not say which control-file fields pg_rewind now trusts as-is and which it derives from the WAL scan.

Precedent: [Fix pg_rewind false positives caused by shutdown-only WAL](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com) (Withdrawn, 2026-05-01)

**Question:** For a target that crashed after promotion but before a checkpoint, which timeline does pg_rewind use, and how is it derived?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. pg_rewind gets a second end-of-WAL rule that must match the server's

An old primary crashes with a torn last segment or an aborted contrecord. The pg_rewind scan stops at a different LSN than crash recovery would, so pg_rewind reports "no rewind required" or misses blocks. The server then refuses the new timeline or runs with stale pages, and the operator finds out only at startup or later.

Today the server decides where an unclean target's WAL ends, and pg_rewind reads the result from a shutdown checkpoint. The patch moves that decision into parsexlog.c, so any later change to end-of-WAL handling or to records with side effects must also be made in pg_rewind. A missed change does not fail loudly. [P8], from the same area and author, was withdrawn after a reviewer said pg_rewind's view of end of WAL cannot differ from the server's, and message #1 does not explain why that concern no longer applies.

Precedent: [Fix pg_rewind false positives caused by shutdown-only WAL](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com) (Withdrawn, 2026-05-01)

**Question:** How does the patch make sure the pg_rewind end-of-WAL point always equals the point crash recovery would reach, and what test fails if the two drift apart?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Stale postmaster.pid removal by PID check alone can expose a live cluster

The data directory is on a shared volume, and a postmaster runs on it in another container or PID namespace. pg_rewind sees no such PID, removes postmaster.pid and rewrites relation files under the running server. The data is corrupted, and nobody notices until that server crashes or returns wrong rows.

Under the old design, a single-user postgres start protected the directory with the server's lock-file logic, which also checks whether the old shared memory segment is still in use. The commit message says pg_rewind removes postmaster.pid "when its PID no longer exists", which is a weaker check. The proposal does not say how it handles PID reuse or postmasters in another namespace.

**Question:** What interlock beyond a PID existence check stops pg_rewind from removing postmaster.pid and writing to a data directory that a postmaster in another namespace still uses?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com); `replication-ondisk-3`</sub>

### 4. Removing --no-ensure-shutdown breaks existing failover scripts

A failover tool such as a cluster manager runs pg_rewind --no-ensure-shutdown after promoting a standby. After the upgrade, pg_rewind exits with an unknown-option error. The old primary is not rejoined, and the operator finds out during the next failover, when the cluster runs without a standby.

The commit message removes the option outright because pg_rewind no longer starts the target. Users who pass it asked for exactly what the new code does: skip the single-user start. Postgres usually keeps such options as accepted no-ops for some releases, or deprecates them first. The touched files include no documentation, so the reference page would still list the option and the clean-shutdown requirement.

**Question:** Why can --no-ensure-shutdown not stay as an accepted no-op, with the pg_rewind documentation updated to match the new behaviour?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com); `standards-compat-1`</sub>

### 5. No field evidence that crash recovery dominates real rewind time

An operator rewinds a crashed old primary during failover. The rewind may finish a few seconds sooner, but every rewind now relies on pg_rewind's own end-of-WAL scan instead of the server's recovery code. If that scan mishandles a case that server recovery handles, the operator finds out when the rewound node fails to start or diverges.

The only evidence is one local benchmark in message #1, and nobody has replied. It saves under 1.5 s for 100 MB and about 19 s for 2000 MB. Message #1 gives no workload, storage, or redo share, and cites no user report that this step slows failover. The commit message drops the dependency on a shutdown checkpoint and minRecoveryPoint, so the new path may also cover clean-shutdown targets, which gain nothing.

**Question:** Which production failovers spend a large share of pg_rewind time in single-user crash recovery, and how many seconds would this save there?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. For a target that crashed after promotion but before its first checkpoint, which timeline and checkpoint does pg_rewind use, and how does it derive them?
2. Which control-file fields does pg_rewind now trust as read, and which does it derive from the WAL scan?
3. Which end-of-recovery actions of the server does the scan not perform, and what test fails if the scan and crash recovery reach different end points?
4. What interlock beyond a PID existence check stops pg_rewind from removing postmaster.pid when the postmaster is dead but orphaned backends still run?
5. Why can --no-ensure-shutdown not stay as an accepted no-op, with the pg_rewind documentation updated to match?

## Touch points

- **on-disk format / pg_upgrade:** pg_rewind reads pg_control and rewrites data files of an unclean target without recovery having updated the control file first.
- **WAL / replication / standby:** pg_rewind now finds the end of target WAL and the pre-divergence checkpoint by its own scan.
- **security / privileges:** see objection replication-ondisk-3: Stale postmaster.pid removal by PID check alone can expose a live cluster
- **maintenance burden ("who maintains this in 10 years"):** pg_rewind takes over work that server crash recovery did, and the patch removes the --no-ensure-shutdown option.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No field evidence that crash recovery dominates real rewind time

## Who would commit it, and when

- Owner: silent. The owner needs to know crash recovery, xlogreader and timeline handling as well as pg_rewind. Nobody has replied in the visible thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. Rewinding a crashed old primary is a common use of pg_rewind, and the benchmark shows 25% to 37% shorter runs. The patch is about 27 KiB in pg_rewind only, but review needs recovery knowledge, and the control-file and interlock questions come before code review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
