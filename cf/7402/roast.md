# Design pre-review: Reduce SyncRepLock contention on the commit path

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7402](https://commitfest.postgresql.org/patch/7402/); [thread](https://www.postgresql.org/message-id/flat/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal, and two setups measured gains of about 9% and 4.6%. The remaining concerns are fixable design points. In 0003, a FATAL or proc_exit may drop a pending release (replication-ondisk-1), and five separate exit sites must each remember it (maintenance-fit-1). In 0004, a measured saving of about 2% pays for a new atomic (maintenance-fit-2), which the author already offered to drop. Signing off on the 0003 exit paths needs a committer who knows walsender shutdown.

Not checked: The code was not read, so the 0003 exit paths, the lock-free argument in 0002 and 0004, and the stats.sql change in 0005 are unchecked. Only 3 of 7 messages were shown.

## Prior attempts

None found before 2026-08-17.

## Objections a reviewer is likely to raise

### 1. Deferred release in 0003 may be lost on FATAL or proc_exit paths

A walsender stores the standby's flush position from a drained reply and marks a release as pending. The process then leaves through a FATAL or a proc_exit, for example pg_terminate_backend or postmaster death, before the drain ends. With one synchronous standby, committers whose WAL that standby already flushed keep waiting until a walsender sends a later reply.

The 0003 commit message covers four explicit returns, plus ERROR through WalSndErrorCleanup(). FATAL and proc_exit do not go through WalSndErrorCleanup(), and WalSndKill() clears the slot positions, so the acknowledgement is gone once the walsender exits. In message #1 the author asks whether an exit path was missed, and no outside reviewer has answered (messages #2 to #7). Today the release runs per reply, so this window does not exist.

**Question:** Does every way a walsender can leave the reply drain, including FATAL, proc_exit and postmaster death, either run the pending release or provably drop nothing a committer waits for?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/3024af42-d264-4bb8-aa58-377b286bd4b3%40postgrespro.ru), [#7](https://www.postgresql.org/message-id/CANeUpr-5B7B%2BLnVeWeB2uJ%3DW4%3DfHKL_UahZFcakveMRrdOYhnQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. 0003 adds an exit-path invariant that later walsender changes can break silently

A later patch adds a message type or an early return to the reply drain in walsender.c without running the pending release. The acknowledged committers stay asleep until another reply happens to arrive. If the standby has disconnected, they hang, and the bug shows up as a rare field report of stuck commits.

Message #1 asks whether any exit path was missed, and no third reviewer has answered. The commit message lists goodbye, EOF, invalid type, unexpected type and WalSndErrorCleanup() as separate places that each must remember the release. The tests in 007_sync_rep.pl cover only today's paths and skip on Windows (message #7). So the rule depends on future authors knowing about it, and nothing fails loudly when one does not.

**Question:** Can the pending release run from one place that every exit from the drain passes through, so that a new exit path cannot skip it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CANeUpr_1LF1rEjPq3fEfRUSCdFkeivQv%2BGhcXSLnze4Ktq4BDA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CANeUpr-5B7B%2BLnVeWeB2uJ%3DW4%3DfHKL_UahZFcakveMRrdOYhnQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. 0004 adds lock-free shared state for a gain of about 2%

On a platform with emulated 64-bit atomics, each synchronous commit reads lsn[] through a spinlock before it takes SyncRepLock, and the fast path almost never hits. Later changes to syncrep.c must keep the lock-free ordering argument correct for every lsn[] access.

Message #1 measured the fast path saving 1.6-2.3% of backend acquisitions, and the author planned to drop 0004. Message #2 kept it on the grounds that it "doesn't cost anything", but no measurement supports that. The commit message says that cost on platforms without native 64-bit atomics is untested. Ten such patches would leave many lock-free reads whose correctness each depends on a monotonicity argument, for little measured gain.

**Question:** Which workload makes the 0004 fast path hit often enough to justify a permanent atomic in WalSndCtlData, and what does it cost with emulated atomics?

<sub>fixable, level 1b, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CANeUpr_dfdQQK6joN8STmpXy4N8cK2H_mK5cM6grwT%2BM28wzXA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/3024af42-d264-4bb8-aa58-377b286bd4b3%40postgrespro.ru), [#3](https://www.postgresql.org/message-id/CANeUpr9_gec4Vmck3zY%2Baw8g%3Dbfs1KvbbH_gOrisjGM1UeyA_g%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Does a walsender that exits through FATAL, proc_exit or postmaster death during the reply drain run the pending release, or provably drop nothing a committer waits for?
2. Can the pending release run from one place that every exit from the reply drain passes through, so that a new early return cannot skip it?
3. Which workload or synchronous_commit level makes the 0004 fast path hit often enough to justify the atomic lsn[] in WalSndCtlData?
4. What does the 0004 read cost per commit on a platform where 64-bit atomics are emulated with a spinlock?
5. Why does 0005 change stats.sql and stats.out, which run in every regression run?

## Touch points

- **WAL / replication / standby:** 0001-0004 change when and how synchronous replication waiters are released in syncrep.c and walsender.c.
- **ABI / extension API:** 0004 changes WalSndCtlData lsn[] to a 64-bit atomic in walsender_private.h, a private header.
- **maintenance burden ("who maintains this in 10 years"):** 0003 adds a rule that every exit from the reply drain runs the pending release; 0004 adds lock-free shared state.
- **performance on the hot path:** SyncRepWaitForLSN() runs on every commit that wrote WAL; 0004 adds an atomic read there before the lock.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the walsender exit paths and the synchronous replication wait queue. Only the two authors have posted in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost medium. Users of synchronous replication at high commit rates gain about 5 to 9% throughput, and others pay nothing. The patches are small but change concurrency in syncrep.c and walsender.c, and 0003 adds an exit-path rule. 0004 can be dropped or split off without loss to 0001-0003.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
