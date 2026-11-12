# Design pre-review: Fix XLogFileReadAnyTLI silently applying divergent WAL from wrong timeline

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6525](https://commitfest.postgresql.org/patch/6525/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Fix XLogFileReadAnyTLI silently applying divergent WAL from wrong timeline](https://www.postgresql.org/message-id/flat/85386EF6-16B7-4D62-86BE-526A10F93825%40yandex-team.ru), started 2026-02-20
- Messages read: 12 of 12
- Patch prose read: v4-0001-Prevent-archive-recovery-from-reading-divergent-WAL.patch, 0001-Fix-archive-recovery-falling-back-to-wrong-timeline-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated and the demand is shown: a production switchover in message #1, an earlier report in message #2, and a reproduction back to v12 in message #5. One fixable level 2 objection survived, replication-ondisk-1: the back-patched wait is explained only at DEBUG1 and no text tells operators what to do when the segment never arrives. A committer who knows recovery must sign for a behaviour change in back branches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether the new rule is correct when several switch points fall in one segment, or when the segment exists only in pg_wal, could not be judged.

## The proposal, restated

**Problem.** When the switch-point segment of the target timeline is not yet archived, XLogFileReadAnyTLI() falls back to the parent timeline's segment with the same number. That copy holds WAL from the old primary after the switch, so a standby replays divergent data without an error and can no longer reach the target timeline. The author hit this during a planned HA switchover.

**Proposed design.** After it skips timelines that began in a later segment, XLogFileReadAnyTLI() tries only the newest eligible timeline for the segment. If that file is in neither the archive nor pg_wal, recovery waits for it and does not read a parent-timeline copy. This replaces the old targetBeginSeg check, which guarded only the target timeline's own switch point and missed intermediate timelines.

**User-visible surface.**
- Archive recovery now waits for the switch-point segment of the owning timeline where it used to replay the parent timeline's copy.
- New DEBUG1 message "not searching older timelines for WAL segment" when recovery declines to fall back.

**What the author claims.**
- Fixes silent replay of divergent WAL, including a three-timeline case the old guard missed.
- A visible stall is better than silently applying divergent data, even for setups that rely on the fallback.
- Ants Aasma's earlier patch from a linked thread is functionally equivalent.
- A reviewer reports the bug reproduces back to v12; the commit message targets back-patch through 14.
- Two new TAP tests cover the two- and three-timeline cases and check that recovery finishes once the correct segment appears.

**Thread state.** Surya Poondla and solai v reviewed v4 and call it ready for a committer; Surya still asks that the DEBUG1 message fire only when an older timeline exists. The separate walreceiver state patch (0002/0003) was disputed by Stepan Tyagushev and dropped from v4, to move to its own thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-02-20; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patched stall can leave existing standbys waiting with no visible log message

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `replication-ondisk-1`

A standby on v14-v18 takes a minor update. Later a cascading failover promotes a child timeline that dies before archiving its switch-point segment. Before the update, recovery replayed the parent copy and went on; after it, recovery waits forever, and the only sign at default log levels is that recovery stops moving.

Message #5 names this case and accepts the stall as the right default, but asks that it be stated. The v4 commit message does not mention it, and the patch adds no docs or release-note text. Message #10 keeps the only explanation at DEBUG1 and sends it on every retry, including on single-timeline standbys (message #11). In a minor release, a committer would likely want users to be able to tell this new wait from a hang, and to know what to do about it.

Thread: [#5 (surya poondla, 2026-07-02)](https://www.postgresql.org/message-id/CAOVWO5oaVqcm54w9LmY6JSx5Mk_XwUc4iJPDO8ZZYnHLoVX5JA%40mail.gmail.com), [#10 (Andrey Borodin, 2026-09-04)](https://www.postgresql.org/message-id/3238B48C-91CC-4029-AEEB-59AC561A25B0%40yandex-team.ru), [#11 (surya poondla, 2026-09-08)](https://www.postgresql.org/message-id/CAOVWO5rp%3Dxu_VUNDXzXXCc7bih2H8_NGct5U10-9pafef%3DSxMw%40mail.gmail.com)

**Question:** How does an operator on a back branch learn that recovery is waiting for the owning timeline's switch-point segment, and what are they told to do when that segment will never be archived?

Counter-check: held. Message #5 accepts the stall but asks that the commit message state the case, and v4 does not. Message #10 keeps the only explanation at DEBUG1, and no text tells back-branch operators what to do when the segment never arrives.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes which WAL segment file archive recovery reads at a timeline switch point in xlogrecovery.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The v4 commit message targets back-patch through 14, and recovery then waits where it used to continue. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch changes the timeline fallback rule in XLogFileReadAnyTLI() and adds two TAP tests in src/test/recovery. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does an operator on a back branch learn, at the default log level, that recovery is waiting for the owning timeline's switch-point segment?
2. What should an operator do when the child timeline died before archiving its switch-point segment, so the segment will never arrive?
3. Will the commit message and release notes state the cascading-failover case from message #5, where recovery now waits and used to continue?
4. Will the DEBUG1 message "not searching older timelines for WAL segment" fire only when an older timeline exists, as the reviewer asks?
5. Why did the 2022 thread with the functionally equivalent patch end without a commit, and does that reason apply here?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows archive recovery and timeline switching in xlogrecovery.c, and who is willing to back-patch a behaviour change. No committer has spoken in the visible thread.
- **Liability:** The owner signs for a recovery behaviour change in every supported branch from 14 on. Any standby that waits after a minor update where it used to continue becomes the committer's report to answer, for the life of those branches.
- **Author follow-up:** The author produced four versions, dropped the disputed walreceiver patches from v4, and plans separate threads for two related issues.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The bug causes silent replay of divergent WAL for standbys that use a WAL archive across a timeline switch, and two reviewers call v4 ready for a committer. The patch is about 14 KiB in one recovery file plus tests, but no committer is engaged and back-patching needs care.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back-patched recovery stall has no visible signal or escape path. Refuted (not specific): Duplicate of replication-ondisk-1. This version also says the operator has no setting to get past the stall, which the material does not establish.
- `maintenance-fit-1` Back-patched stall is visible only at DEBUG1 and has no documented way out. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same back-branch stall and missing operator guidance and also covers the DEBUG1 message firing on single-timeline standbys.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches: [P1] archives history files and [P2] renames .partial segments, and neither recorded reason applies to this proposal. Messages #2 and #5 point to a 2022 thread with an equivalent patch, which message #5 says this patch follows, but the material does not say why that thread ended without a commit.
- standards-compat: 1 objection(s). The proposal adds no syntax, GUC, catalog object or pg_dump state. The only compatibility issue from this angle is the behaviour change in back branches.
- replication-ondisk: 1 objection(s). The core change keeps recovery off divergent WAL and adds no on-disk, catalog or protocol state. The only concern from this angle is how the behaviour change reaches back branches.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which archived WAL file the startup process reads during recovery, and only superusers or the operator control that through restore_command, the archive and the timeline history; no new role, function, view or setting is involved. The new DEBUG1 message appears only at a debug log level and reveals no data beyond segment names.
- maintenance-fit: 1 objection(s). The fix belongs in core and adds no hot-path cost, so the angle finds no fit problem. No committer has engaged in the visible thread yet. That limits committability, but it is no objection to the design.
- already-exists: 0 objection(s). This is a bug fix in XLogFileReadAnyTLI(). No existing setting, view or extension stops archive recovery from reading the parent timeline's copy of a switch-point segment, and an external tool cannot change which file recovery opens. The fix narrows the existing loop and adds no second mechanism.
- demand: 0 objection(s). The demand is real. This is a bugfix for silent replay of divergent WAL, seen in a production HA switchover (message #1). Ants Aasma reported the same bug in an earlier thread (message #2), and a reviewer reproduced it back to v12 (message #5). The cost falls on recovery only at segments it cannot find, and the thread weighed the changed stall behaviour (messages #1, #5).

### Precedents retrieved

- P1: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - used in the review; reason summarised from its thread
- P2: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - used in the review; reason summarised from its thread
- P3: [get current and recovery timeline](https://commitfest.postgresql.org/patch/2225/) (cf, Returned with feedback, 2020-01-31) - judged not relevant; reason summarised from its thread
- P4: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P5: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P6: [Logical decoding timeline following take II](https://commitfest.postgresql.org/patch/779/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P7: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P8: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P9: [Allow providing restore_command as a command line option to pg_rewind](https://commitfest.postgresql.org/patch/3213/) (cf, Rejected, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant; reason summarised from its thread
- P11: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P12: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P13: [Allow archive_mode to be changed without server restart?](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Point-In-Time_Recovery_.28PITR.29) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P14: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P15: [Switching XLog source from archive to streaming when primary available](https://commitfest.postgresql.org/patch/3663/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P16: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P17: [Revert commit 66c0185a3 and follow-on patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7204f35919b7e021e8d1bc9f2d76fd6bfcdd2070) (revert, Reverted, 2024-05-20) - judged not relevant
- P18: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P19: [Creating backup history files for backups taken from standbys](https://commitfest.postgresql.org/patch/1231/) (cf, Returned with feedback, 2018-03-05) - judged not relevant
- P20: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P21: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant
- P22: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P23: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2935/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P24: [tracking last allocated XLOG segment in control data](https://commitfest.postgresql.org/patch/444/) (cf, Rejected, 2016-01-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 98 s; cost reported by the CLI: $1.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:53:47.925Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
