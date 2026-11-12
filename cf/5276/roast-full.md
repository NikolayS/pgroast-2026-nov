# Design pre-review: Fix race between WAL flush and InstallXLogFileSegment()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5276](https://commitfest.postgresql.org/patch/5276/) (Needs review); authors: Thomas Munro (macdice)
- Thread: [InstallXLogFileSegment() vs concurrent WAL flush](https://www.postgresql.org/message-id/flat/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), started 2024-02-02
- Messages read: 6 of 6
- Patch prose read: 0001-Fix-InstallXLogFileSegment-concurrency-bug.patch, 0002-Track-end-of-installed-WAL-space-in-shared-memory.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the fix itself: the window between rename() and the parent directory fsync is stated plainly and affects every installation that writes WAL. The two surviving objections, replication-ondisk-1 and maintenance-fit-1, are fixable design points about 0002. They ask how the tracked segment number stays a safe bound across timeline switches, and whether 0002 is needed without a measurement. A committer who knows WAL file handling can settle both.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check where the tracked value is updated, how it handles timeline switches, or which lock mode and hold time 0001 uses. Messages #4 to #6 were not shown in full.

## The proposal, restated

**Problem.** Postgres creates WAL space by renaming a file into place, and there is a window between rename() and fsync of the parent directory. A backend can open the new name, write and fdatasync WAL, and then a power loss can drop the rename. Recovery then cannot find WAL that was flushed, which can break the log-before-data rule or lose a commit already reported to a client.

**Proposed design.** Patch 0001 makes XLogFileInitInternal() wait until InstallXLogFileSegment() has made the new name durable, using ControlFileLock. Patch 0002 tracks the last known installed segment in shared memory, so a backend opening an earlier segment can skip the lock. A later version starts that value at the segment of the starting checkpoint.

**What the author claims.**
- The data loss window comes from reading the code, and the author has not seen it happen.
- Breakage would need very bad luck, and the author sees no existing defence against it.
- Waiting on every install is pessimistic, and 0002 avoids most of the waits.
- The author is not sure 0002 is worth it, or whether a fsync before returning the fd is the better fix.

**Thread state.** Yugo Nagata proposed an fsync before returning the fd, and Kyotaro Horiguchi and Soumyadeep Chakraborty preferred the lock approach with 0002. Soumyadeep posted v3 and asked whether other durable_rename() callers, such as the WAL summarizer, have the same window. Soumya S Murali reported regression tests pass, and the CommitFest entry still needs review and a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-02-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Skip-the-lock value tracks segment number only, but names carry timeline

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

A standby is promoted, and end of recovery installs the last segment under the new timeline's file name. A backend opens that segment and skips the lock because its segment number is not above last_known_installed_segno, which was seeded from the starting checkpoint. It writes and flushes commits, then power loss drops the rename and recovery cannot find them.

0002 lets a backend skip waiting when the segment it opens is not later than the last known installed segment (message #1). v3 seeds that value from the starting checkpoint (message #5), and message #6 calls this a conservative lower bound. A WAL file name combines timeline and segment number, and promotion installs a file whose segment number may already lie at or below that bound. The material does not say whether a timeline switch resets or bypasses the tracked value.

Thread: [#1 (Thomas Munro, 2024-02-02)](https://www.postgresql.org/message-id/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), [#4 (Kyotaro Horiguchi, 2024-02-06)](https://www.postgresql.org/message-id/20240206.165816.1496814607463556734.horikyota.ntt%40gmail.com), [#5 (Soumyadeep Chakraborty, 2024-10-31)](https://www.postgresql.org/message-id/CAE-ML%2B-vkbp_oU8qUdJux01EjQK1sDic48yT_aEPmoA0yC8mxw%40mail.gmail.com), [#6 (Soumya S Murali, 2026-01-14)](https://www.postgresql.org/message-id/CAMtXxw_U1N6wDrPO1UyYOMV63YZuehM12DW0Nm%2BTbMSB3fDdEg%40mail.gmail.com)

**Question:** After a timeline switch at promotion, can any backend skip the lock while opening a new-timeline segment whose segment number is at or below last_known_installed_segno?

Counter-check: held. The stated scenario is weak, because the startup process finishes the end-of-recovery install before any backend may write WAL. But the tracked value holds a segment number only, and the thread does not say how recycling into new-timeline names after promotion stays safe.

### 2. Shared-memory install marker in 0002 is a silent invariant for future paths

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A later patch adds or changes a path that installs WAL segments, such as promotion, timeline switch or restore from archive, and does not update last_known_installed_segno, or sets it too high. A backend then skips ControlFileLock for a segment whose name is not yet durable. The original window reopens, and nobody notices until a power loss makes flushed WAL unreachable.

0002 makes correctness depend on a value that every installer must keep as a safe bound, and an error in that value causes no test failure. The thread checks the starting-checkpoint initialization (messages #4, #5, #6) but does not cover standby or archive recovery paths, or segments with the same segno on a new timeline. The author says 0002 may not be worth it (message #1), and no measurement shows the lock wait from 0001 alone costs anything.

Thread: [#1 (Thomas Munro, 2024-02-02)](https://www.postgresql.org/message-id/CA%2BhUKGLO02j2WLiQ73iZ%2BCEY1G%2BLPmHo3PXaYTaFY9Hj222mEQ%40mail.gmail.com), [#4 (Kyotaro Horiguchi, 2024-02-06)](https://www.postgresql.org/message-id/20240206.165816.1496814607463556734.horikyota.ntt%40gmail.com), [#5 (Soumyadeep Chakraborty, 2024-10-31)](https://www.postgresql.org/message-id/CAE-ML%2B-vkbp_oU8qUdJux01EjQK1sDic48yT_aEPmoA0yC8mxw%40mail.gmail.com), [#6 (Soumya S Murali, 2026-01-14)](https://www.postgresql.org/message-id/CAMtXxw_U1N6wDrPO1UyYOMV63YZuehM12DW0Nm%2BTbMSB3fDdEg%40mail.gmail.com)

**Question:** Which code paths install WAL segments, why does each keep last_known_installed_segno a safe lower bound, and what measurement shows 0001 alone is too slow?

Counter-check: held. Segment installs go through InstallXLogFileSegment(), so the risk of a forgotten update is smaller than stated. But no measurement shows that 0001 alone is too slow, and the author doubts 0002 is worth the extra invariant.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-1: Skip-the-lock value tracks segment number only, but names carry timeline |
| WAL / replication / standby | **yes** | Both patches change how xlog.c installs and opens WAL segment files, on primary and standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but the thread does not say whether to back-patch; 0002 adds a shared-memory field. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds a shared-memory value that every segment install path must keep as a safe lower bound. |
| performance on the hot path | **yes** | 0001 makes XLogFileInitInternal() wait on ControlFileLock when a backend opens a new WAL segment. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. After a timeline switch, can a backend skip the lock while opening a new-timeline segment whose segment number is at or below last_known_installed_segno?
2. What measurement shows that 0001 alone, with the wait on ControlFileLock at each segment open, is too slow to commit without 0002?
3. Why is the lock wait preferred over Yugo Nagata's fsync of the directory before the fd is returned, and what does each cost at a segment switch?
4. Is the fix meant for back-patching, and if so, does 0001 alone go to the supported branches?
5. Does a test or fault injection show that flushed WAL becomes unreachable when the rename is lost, and that the patch closes that case?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The review notes record that a committer wrote the patch; the final push still needs someone who knows WAL segment recycling, timeline switches and ControlFileLock usage.
- **Liability:** The owner signs for WAL durability and for any stall at segment switches on all supported platforms, with no end date. With 0002, the owner also signs for the tracked value staying a safe bound as install paths change.
- **Author follow-up:** The author answered in message #3 on the day of posting. The shown messages give no later sign, and the entry needs a rebase since 2026-04.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** most users
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The fix guards flushed WAL for every installation, and both patches are small and confined to xlog.c. Review cost is in the crash-safety reasoning; a rebase and a decision on 0001 alone versus 0001 plus 0002 would unblock it.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Standby flush acknowledgements are not shown to be covered. Refuted (wrong facts): The walreceiver creates segments through XLogFileInit(), which calls XLogFileInitInternal(), the function that 0001 makes wait. So the walreceiver's writes get the same wait as a backend's.
- `maintenance-fit-2` Point fix leaves the durable_rename() window for other callers. Refuted (no failure scenario): The scenario is a future durable_rename() caller, and that window exists today with or without this patch. The patch fixes the WAL segment case and makes no other caller worse, so the scenario is not a failure of this proposal.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] reverted a switch from durable_rename_excl() to durable_rename() because a TLI history file could be overwritten. That reason does not apply to closing the rename/fsync window in WAL segment install. The thread cites no earlier discussion of the idea.
- standards-compat: 0 objection(s). The proposal has no user-facing contract to check: no SQL syntax, GUC, catalog object, pg_dump output or psql change, and both patches touch only xlog.c. The open questions are lock cost, whether the window can be shown, other durable_rename() callers and back-patching, and those belong to the safety and performance angles. Back-patch suitability cannot be judged without the code, because the new shared-memory counter's ABI effect depends on where it is placed.
- replication-ondisk: 2 objection(s). The proposal changes no on-disk format, WAL record, catalog or protocol. Both objections ask the design to show it covers every path that installs a segment and then writes to it. Neither questions the fix itself.
- security: 0 objection(s). The proposal is an internal WAL durability fix with no privilege or exposure aspect. It adds no user-callable interface, privilege, setting or monitoring output, and the new waits on ControlFileLock happen only inside server WAL writing, which an unprivileged user cannot drive beyond normal write load.
- maintenance-fit: 2 objection(s). Neither objection is against the fix itself. The bug is real, the patch is small, and a committer wrote it, so these are requests to tighten 0002 and the durable_rename() contract.
- already-exists: 0 objection(s). This is a crash-safety bugfix inside xlog.c. No existing feature, setting, extension or external tool closes the window between rename() and the parent directory fsync for WAL segments. Yugo Nagata's alternative (an fsync before the fd is returned) is a design choice inside the same fix, and other reviewers discuss it; this angle does not cover it. [P1] is about durable_rename_excl() and timeline history files, so it does not apply.
- demand: 0 objection(s). The demand angle has no objection here, because this is a durability bugfix and its users are every Postgres installation that writes WAL. A possible loss of a commit already reported to the client does not need field reports to justify it; the missing reproduction and cost measurements belong to the design and safety angles.

### Precedents retrieved

- P1: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - used in the review
- P2: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P3: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [Harden against premature discarding of WAL segments in TAP tests](https://commitfest.postgresql.org/patch/3333/) (cf, Withdrawn, 2021-11-03) - judged not relevant; reason summarised from its thread
- P5: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P6: [Add support for --end-segment-command in pg_receivewal](https://commitfest.postgresql.org/patch/1040/) (cf, Returned with feedback, 2017-03-21) - judged not relevant; reason summarised from its thread
- P7: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant; reason summarised from its thread
- P8: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P9: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P10: [Atomic rename feature for Windows](https://commitfest.postgresql.org/patch/3347/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P11: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P12: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P13: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P14: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P15: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant
- P16: [pgrename_temp() – concurrent rename of temporary files on Windows](https://commitfest.postgresql.org/patch/2230/) (cf, Returned with feedback, 2020-04-05) - judged not relevant
- P17: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P18: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant
- P19: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant
- P20: [REINDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1060/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P21: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P22: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - judged not relevant
- P23: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P24: [Fix calculations on WAL recycling](https://commitfest.postgresql.org/patch/1730/) (cf, Rejected, 2018-07-23) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 142 s; cost reported by the CLI: $1.41
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 62 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:31:04.228Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
