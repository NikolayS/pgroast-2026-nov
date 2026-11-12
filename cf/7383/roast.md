# Design pre-review: Throwing away unnecessary spin-locks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7383](https://commitfest.postgresql.org/patch/7383/); [thread](https://www.postgresql.org/message-id/flat/CAE8JnxPcmZVDkf39Pa%3DuBGcVacsm6OkGxYigp6Znxu%3DZ3_DE7w%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Level 1b fails on demand-1. The thread names no workload that spends time on ckpt_lck, message #10 says contention was not measured, and message #13 expects no performance effect from v2. The lock is taken a few times per checkpoint request, so the new lock-free protocol carries ordering risk with no stated gain. The level 2 objection replication-ondisk-1 survived but was not evaluated because of the base-up rule.

Not checked: The code was not read, so the version-number scheme, its bit layout, the xlog.h change and any ordering comments in checkpointer.c could not be checked. Only 3 of 13 messages were shown in full.

## Prior attempts

- [fix spinlock contention in LogwrtResult](https://www.postgresql.org/message-id/flat/20200831182156.GA3983@alvherre.pgsql) (similar idea; Withdrawn, 2022-09-23). "I realized that because of commit e369f3708636 this change is no longer as critical as it used to be, so I'm withdrawing this patch from the commitfest." ([message](https://www.postgresql.org/message-id/20220923084941.g3756xqz6lwnqwso%40alvherre.pgsql)) Different now: P1 targeted a contended lock, info_lck on LogwrtResult, and converted its fields to 64-bit atomics. v2 here targets ckpt_lck, with no contention claim and no expected performance effect.
- [Move PinBuffer and UnpinBuffer to atomics](https://www.postgresql.org/message-id/flat/2400449.GjM57CE0Yg@dinodell) (similar idea; Returned with feedback, 2015-12-16). "Which you made ineffective because PinBuffer() doesn't take a lock anymore. Mutual exclusion through locks only works if all participants take the locks." ([message](https://www.postgresql.org/message-id/20150914154142.GB9666%40alap3.anarazel.de)) Different now: P7 removed the lock on one side while other paths still relied on it, which matches the objection to v1 writes in message #8. v2 removes ckpt_lck from all participants, so that reason does not apply to v2.

## Objections a reviewer is likely to raise

### 1. No measured contention or user problem justifies lock-free conversion

A committer pushes the lock-free checkpointer protocol to gain nothing measurable. Later, a memory-ordering bug on ARM or POWER makes RequestCheckpoint with CHECKPOINT_WAIT return early or report the wrong failure. Users find out from an unexplained checkpoint error or a backup that did not wait.

The thread names no workload, user report or benchmark where these spinlocks cost anything. In message #10 the author says contention was not measured, and in message #13 expects v2 to have no performance effect. The checkpointer locks are taken a few times per checkpoint, which is a cold path. Every user carries the risk of subtle ordering bugs, as message #8 warns, while nobody is shown to gain. [P1] pulled a similar atomics change for the same mix of hard barrier rules and reduced need.

Precedent: [fix spinlock contention in LogwrtResult](https://www.postgresql.org/message-id/flat/20200831182156.GA3983@alvherre.pgsql) (Withdrawn, 2022-09-23)

**Question:** Which workload shows measurable time spent on ckpt_lck or on another spinlock you plan to convert, and how much does the change recover?

<sub>fixable, level 1b, demand; messages [#8](https://www.postgresql.org/message-id/yvmmtoinl655fpdbravy6kqti4d3yeqaclhywqzqm4xsklmnlz%40qejqsnhxipa4), [#9](https://www.postgresql.org/message-id/CAHyXU0yjziQUN1FJTtxQf3AD6hkUG2W8OS0Omm4uE%3DuBzfYs4Q%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAE8JnxPmkOvmVWPcycKgCsG9ryQFNZSFS%2BUR030bQxGC-rZg0A%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/CAHyXU0yEAk184baUAb-K%2BdV3eQ2D2gkTkPRKvLrtg4zusmYQSw%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAE8JnxP%3DGBWwc3%2BC4YJ-OctCiNf0Y_0Y3Ph%2BjW-QojHag-v92Q%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. What benefit does removing ckpt_lck bring, given that message #13 expects no performance effect from v2?
2. Which workload shows measurable time spent on ckpt_lck or on another spinlock you plan to convert, and how much does the change recover?
3. What argument, written in checkpointer.c, shows that a CHECKPOINT_WAIT caller on ARM or POWER never returns before a checkpoint that saw its flags has completed?
4. Is the 20-file v1 series withdrawn, or is it still proposed for the files other than checkpointer.c?
5. What does v2 change in src/include/access/xlog.h, and does it affect the CHECKPOINT_* flag values that extensions pass to RequestCheckpoint?

## Touch points

- **WAL / replication / standby:** v2 changes how RequestCheckpoint with CHECKPOINT_WAIT detects that its checkpoint finished, and callers rely on that for durability.
- **ABI / extension API:** v2 changes src/include/access/xlog.h without saying how; v1 adds slock_* helpers to storage/spin.h.
- **maintenance burden ("who maintains this in 10 years"):** v2 replaces a spinlock with a version-number protocol whose memory-ordering rules later editors of checkpointer.c must keep.
- **performance on the hot path:** v1 touches lwlock.c, freelist.c and xlog.c; v2 touches only the checkpoint request path, which is cold.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No measured contention or user problem justifies lock-free conversion

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund replied in messages #2, #8 and #12 but said he cannot review many rounds, and v2 has no replies. The owner must know checkpointer.c and the memory-barrier rules on weakly ordered platforms.
- Not now, show demand first: A benchmark or profile that shows time spent on ckpt_lck, or on another spinlock chosen for conversion, and the gain after the change. A stated simplification of checkpointer.c that reviewers accept as worth the new ordering rules would also count.
- Priority: park; audience unclear; review cost medium. No user or workload is shown to gain, and the author expects no performance effect from v2. The patch is one file, but the patch code was not read, and its lock-free protocol needs a reviewer who can check memory ordering.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
