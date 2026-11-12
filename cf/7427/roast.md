# Design pre-review: [RFC PATCH v1] On-demand WAL replay: accept connections before crash recovery has applied the WAL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7427](https://commitfest.postgresql.org/patch/7427/); [thread](https://www.postgresql.org/message-id/flat/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is stated and measured, but the design leaves the pending window undefined in several places. No checkpoint can complete during the drain and nothing bounds pg_wal growth (replication-ondisk-1). The proposal does not cover a redo PANIC in a backend (replication-ondisk-2), and nothing checks the new rules for redo routines (maintenance-fit-1). The simpler unwaited-checkpoint path is not compared (already-exists-1). All are fixable, and the thread has only the author's message.

Not checked: Without the code, nobody can confirm that no path makes a page visible before it is current, or that failed replay cleans up correctly. The cost in StartReadBuffersImpl() with the setting off is also unmeasured.

## Prior attempts

- [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://www.postgresql.org/message-id/flat/CALj2ACVvy+06hv7HocOFAHEBZWxLbc-t_61axdCUq9NaR2-c0Q@mail.gmail.com) (related; Withdrawn, 2022-02-27). "This patch would bring some extra visibility for something that's not incorrect either on HEAD, as end-of-recovery checkpoints are the same things as shutdown checkpoints." ([message](https://www.postgresql.org/message-id/YhmaUxDMemv5hmba%40paquier.xyz)) Different now: P14 changed the control file state around the end-of-recovery checkpoint. This proposal opens the server before any such checkpoint, so code that treats it as a shutdown checkpoint is affected.

## Objections a reviewer is likely to raise

### 1. No checkpoint during the drain lets pg_wal and the redo window grow

A busy primary reopens after a crash and takes full write load while the worker drains for minutes. With CreateCheckPoint() refused, max_wal_size is not honoured and pg_wal can fill the disk, causing a PANIC. Any second crash replays from the old redo point, through all old and new WAL.

Message #1 refuses checkpoints until the LSN index is gone, and case C shows a 507 s drain on an idle server. Under load, the worker competes with clients, so the window has no stated upper bound. The proposal has no limit on WAL kept, no cap on drain time, and no fallback that forces a checkpoint. [P9] was reverted because committed changes stayed un-checkpointed with full reliance on the WAL stream, and this design creates the same state after every crash.

Precedent: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (Reverted, 2012-06-13)

**Question:** What bounds pg_wal growth and the redo window while the worker drains under write load, and what happens when that bound is reached?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Redo errors in backends can turn into a crash-restart loop

One page has a record whose redo hits a PANIC-level consistency check. In stock recovery, startup stops once with a clear message. Here the server opens, the first reader of that page triggers a cluster-wide crash restart, recovery reopens, and the next reader of the page crashes it again, each time over more WAL.

Many redo routines report inconsistent pages with PANIC, and the commit message covers only ERROR and FATAL cleanup in a backend. Moving redo into ordinary backends and the worker means a redo PANIC now resets all of shared memory, including the DSM index, while clients are connected. The proposal does not say how a page that fails replay is reported, isolated, or kept from restarting the cluster again and again.

**Question:** What happens when on-demand redo of one page raises PANIC, and how does the design stop repeated crash restarts over a growing WAL window?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. Every redo routine, including custom rmgrs, gains unchecked new rules

A later patch, or an extension using a custom WAL resource manager, adds a redo routine that reads a second page or assumes all blocks of a record are applied together. With fast_crash_recovery on, that page is replayed wrongly or nests replay. The damage shows up after some production crash, as silent page corruption.

Message #1 makes correctness depend on new rules: redo of an indexed page never reads another indexed page, other blocks report BLK_DONE, initialized blocks go to a scratch local buffer, and an exclusive lock stands in for a cleanup lock. These hold for today's core redo routines by inspection, but nothing in the design checks them. The default is off and only crash tests exercise the feature, so a routine that breaks a rule passes check-world. Custom rmgrs are outside the project's control.

**Question:** What check, assertion or test fails when a new or extension redo routine breaks the one-page, no-nesting rule this design depends on?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. The existing promotion path already skips the waited end-of-recovery checkpoint

An operator enables fast_crash_recovery and gets a deferred-replay buffer manager path and a new auxiliary process. Most of their downtime was the end-of-recovery checkpoint. Writing an end-of-recovery record and requesting the checkpoint without waiting, as promotion already does, would have removed that part with far less new state.

In case A the checkpoint is 9.2 of 15.1 s, and in B it is 30.4 of 96.1 s. Postgres already ends archive recovery at promotion with an end-of-recovery record and an unwaited checkpoint. The proposal does not compare against reusing that path for crash recovery. Only the gain beyond that simpler change justifies replay inside StartReadBuffersImpl() and the failing CHECKPOINT window.

**Question:** How much time to accept connections remains if crash recovery only ends with an end-of-recovery record and an unwaited checkpoint, and does that remainder justify on-demand page replay?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com); `already-exists-1`</sub>

### 5. CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on

After a crash, a backup tool calls pg_backup_start() or pg_basebackup, or a deployment script runs CREATE DATABASE. Each gets "checkpoint request failed" for as long as the drain runs, which was over 8 minutes in run C. The operator finds out when the nightly backup or a provisioning job fails, with an error that does not name the cause.

With the GUC on, commands that work on every server today fail for an unbounded time after a crash. The GUC changes the semantics of user commands, beyond recovery timing. Message #1, item 4, leaves the choice between failing and waiting open and ships the error in v1. The project's own 018_wal_optimize test already trips on this, which suggests external scripts will too. A committer would likely ask for one defined contract and a specific error message before the GUC is exposed.

**Question:** Will CHECKPOINT, pg_backup_start() and CREATE/DROP DATABASE wait during the drain, and if they fail, what errcode and hint tell the operator why and for how long?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6rnoZ%3D5uyEaox_%3DP4hbsX7dw1grm5gMgg0Ar6S5AmqLFA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What bounds pg_wal growth and the redo window while the worker drains under write load, and what does a second crash in that window replay?
2. What happens when on-demand redo of one page raises PANIC in a backend, and how does the design prevent repeated crash restarts on that page?
3. What assertion or test fails when a new redo routine, or one in a custom resource manager, reads a second page during on-demand redo?
4. How much time to accept connections remains if crash recovery only writes an end-of-recovery record and requests an unwaited checkpoint, as promotion does?
5. Will CHECKPOINT, pg_backup_start() and CREATE/DROP DATABASE wait or fail during the drain, and what do smart, fast and immediate shutdown each do?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: No checkpoint during the drain lets pg_wal and the redo window grow
- **WAL / replication / standby:** Page redo moves out of the startup process, and CreateCheckPoint() refuses until the fast recovery worker finishes.
- **ABI / extension API:** Redo routines, including those of custom WAL resource managers, run under new one-page rules; bufmgr.h changes.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on
- **new GUC:** Adds fast_crash_recovery, a boolean set at server start, default off.
- **backpatchability:** see objection standards-compat-1: CHECKPOINT, backups and CREATE DATABASE fail after restart with the GUC on
- **maintenance burden ("who maintains this in 10 years"):** New auxiliary process, DSM index, and rules that every future redo routine must keep, across bufmgr, checkpointer and recovery.
- **performance on the hot path:** Adds a check in StartReadBuffersImpl(), the common read path; no measurement with the setting off.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows both the buffer manager read path and WAL recovery with checkpoints; none has replied in the thread.
- Priority: fix design first; audience unclear; review cost high. A single 94 KiB patch touches bufmgr, checkpointer, postmaster and recovery, and the behaviour of the pending window is still open. The thread cites no field reports, and the feature excludes standby and archive recovery, so the size of the audience is not shown.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
