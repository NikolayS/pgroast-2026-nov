# Design pre-review: Fix race between WAL flush and InstallXLogFileSegment()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5276](https://commitfest.postgresql.org/patch/5276/); [thread](https://www.postgresql.org/message-id/flat/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the fix itself: the window between rename() and the parent directory fsync is stated plainly and affects every installation that writes WAL. The two surviving objections, replication-ondisk-1 and maintenance-fit-1, are fixable design points about 0002. They ask how the tracked segment number stays a safe bound across timeline switches, and whether 0002 is needed without a measurement. A committer who knows WAL file handling can settle both.

Not checked: Without the code, this review could not check where the tracked value is updated, how it handles timeline switches, or which lock mode and hold time 0001 uses. Messages #4 to #6 were not shown in full.

## Prior attempts

None found before 2024-02-02.

## Objections a reviewer is likely to raise

### 1. Skip-the-lock value tracks segment number only, but names carry timeline

A standby is promoted, and end of recovery installs the last segment under the new timeline's file name. A backend opens that segment and skips the lock because its segment number is not above last_known_installed_segno, which was seeded from the starting checkpoint. It writes and flushes commits, then power loss drops the rename and recovery cannot find them.

0002 lets a backend skip waiting when the segment it opens is not later than the last known installed segment (message #1). v3 seeds that value from the starting checkpoint (message #5), and message #6 calls this a conservative lower bound. A WAL file name combines timeline and segment number, and promotion installs a file whose segment number may already lie at or below that bound. The material does not say whether a timeline switch resets or bypasses the tracked value.

**Question:** After a timeline switch at promotion, can any backend skip the lock while opening a new-timeline segment whose segment number is at or below last_known_installed_segno?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/20240206.165816.1496814607463556734.horikyota.ntt%40gmail.com), [#5](https://www.postgresql.org/message-id/CAE-ML%2B-vkbp_oU8qUdJux01EjQK1sDic48yT_aEPmoA0yC8mxw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAMtXxw_U1N6wDrPO1UyYOMV63YZuehM12DW0Nm%2BTbMSB3fDdEg%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Shared-memory install marker in 0002 is a silent invariant for future paths

A later patch adds or changes a path that installs WAL segments, such as promotion, timeline switch or restore from archive, and does not update last_known_installed_segno, or sets it too high. A backend then skips ControlFileLock for a segment whose name is not yet durable. The original window reopens, and nobody notices until a power loss makes flushed WAL unreachable.

0002 makes correctness depend on a value that every installer must keep as a safe bound, and an error in that value causes no test failure. The thread checks the starting-checkpoint initialization (messages #4, #5, #6) but does not cover standby or archive recovery paths, or segments with the same segno on a new timeline. The author says 0002 may not be worth it (message #1), and no measurement shows the lock wait from 0001 alone costs anything.

**Question:** Which code paths install WAL segments, why does each keep last_known_installed_segno a safe lower bound, and what measurement shows 0001 alone is too slow?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/20240206.165816.1496814607463556734.horikyota.ntt%40gmail.com), [#5](https://www.postgresql.org/message-id/CAE-ML%2B-vkbp_oU8qUdJux01EjQK1sDic48yT_aEPmoA0yC8mxw%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAMtXxw_U1N6wDrPO1UyYOMV63YZuehM12DW0Nm%2BTbMSB3fDdEg%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. After a timeline switch, can a backend skip the lock while opening a new-timeline segment whose segment number is at or below last_known_installed_segno?
2. What measurement shows that 0001 alone, with the wait on ControlFileLock at each segment open, is too slow to commit without 0002?
3. Why is the lock wait preferred over Yugo Nagata's fsync of the directory before the fd is returned, and what does each cost at a segment switch?
4. Is the fix meant for back-patching, and if so, does 0001 alone go to the supported branches?
5. Does a test or fault injection show that flushed WAL becomes unreachable when the rename is lost, and that the patch closes that case?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-1: Skip-the-lock value tracks segment number only, but names carry timeline
- **WAL / replication / standby:** Both patches change how xlog.c installs and opens WAL segment files, on primary and standby.
- **backpatchability:** The entry is tagged Bugfix, but the thread does not say whether to back-patch; 0002 adds a shared-memory field.
- **maintenance burden ("who maintains this in 10 years"):** 0002 adds a shared-memory value that every segment install path must keep as a safe lower bound.
- **performance on the hot path:** 0001 makes XLogFileInitInternal() wait on ControlFileLock when a backend opens a new WAL segment.

## Who would commit it, and when

- Owner: qualified committer engaged. The review notes record that a committer wrote the patch; the final push still needs someone who knows WAL segment recycling, timeline switches and ControlFileLock usage.
- Priority: proceed; audience most users; review cost medium. The fix guards flushed WAL for every installation, and both patches are small and confined to xlog.c. Review cost is in the crash-safety reasoning; a rebase and a decision on 0001 alone versus 0001 plus 0002 would unblock it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
