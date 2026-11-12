# Design pre-review: Batch fsyncs when recycling WAL segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7088](https://commitfest.postgresql.org/patch/7088/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [\[Patch\] Batch fsyncs when recycling WAL segments](https://www.postgresql.org/message-id/flat/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), started 2026-08-02
- Messages read: 4 of 4
- Patch prose read: nocfbot-v4-0002-WIP-Batch-fsyncs-when-recycling-WAL-segments.patch, v4-0001-Batch-fsyncs-when-removing-old-WAL-segments.patch, 0001-Batch-fsyncs-when-recycling-WAL-segments-guarded-by-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No objection survived against 0001, which batches only unlinks and does not touch the WAL write path. For 0002, demand-1 and maintenance-fit-3 stand: every number measures background checkpoint cleanup time, and message #3 says it is "not a TPS result". 0002 adds a durability invariant to the WAL write path for that gain, and the author doubts in message #3 that it is worth the plumbing. Confidence is low because only the author has posted and the verdict rests on missing evidence.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked whether the frontier and the active-batch marker cover every segment installer, or whether 0001's retry through the kept archive-status files works when archiving is off. Four level 2 objections against 0002 survived but were not evaluated here.

## The proposal, restated

**Problem.** A checkpoint that removes or recycles many WAL segments pays one pg_wal directory fsync per segment through durable_unlink() or durable_rename(). The author cites an earlier thread calling these fsyncs a real bottleneck. Systems with many segments per checkpoint, or small segment sizes, pay the most.

**Proposed design.** 0001 unlinks all removable segments first, fsyncs pg_wal once, then removes their archive-status files; timeline-switch cleanup stays unbatched. 0002 (WIP) renames recycled segments in a batch, fsyncs the files and pg_wal once, and adds a timeline-qualified durability frontier in XLogCtl. XLogFileInit() fsyncs the segment and pg_wal itself when the frontier does not cover it or a recycle batch is active.

**User-visible surface.**
- New injection point(s) in the WAL recycle path, used by TAP test 058_wal_recycle_durability.pl.

**What the author claims.**
- Recycle batching cut checkpoint cleanup from a median of about 139 ms to 27 ms with about 44 segments recycled (ext4, Azure).
- With 1 MiB segments and 512 MiB recycled, cleanup fell from 1900 ms to 80 ms.
- 0001 alone cut removal of 128 segments from 394 ms to 154 ms; one segment shows no change.
- The write-path fallback should almost never fire because the checkpointer normally stays ahead.
- 0001 keeps the existing LOG-level policy on directory fsync failure and retains archive-status files so a later checkpoint retries.

**Thread state.** Only the author has posted; four messages carry v1 to v4, with 0002 marked WIP and nocfbot. Three reviewers are listed in the CommitFest, but none has replied, and the author's question about a simpler invariant for 0002 is open.

## 2. Prior attempts

### Speed up WAL file removal during recovery (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1530/](https://commitfest.postgresql.org/patch/1530/)
- **Thread:** [https://www.postgresql.org/message-id/flat/0A3221C70F24FB45833433255569204D1F81B0C8@G01JPEXMBYT05](https://www.postgresql.org/message-id/flat/0A3221C70F24FB45833433255569204D1F81B0C8@G01JPEXMBYT05)
- **Date:** decided 2018-09-30; started 2017-11-17
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Kyotaro HORIGUCHI, Michael Paquier, Fujii Masao (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers found that skipping the per-file fsync breaks crash safety for recycled segments: after a crash a renamed file can keep its old name but hold new contents, which brings back problems fixed by 1d4a0ab1. Deleting all files to avoid renames loses WAL recycling and costs performance. The alternatives (moving the work to the checkpointer, or disabling fsync in the durable_* functions around the loop) also drew objections. With no agreed design, Michael Paquier returned the entry with feedback.
- **Reason (quote):** "In short, if a crash happens in the code paths calling RemoveXlogFile with durable = false before fsync'ing pg_wal, then a rename has no guarantee to be durable, so you could finish again with a file that as an old name, but new contents." ([message](https://www.postgresql.org/message-id/20180221075250.GB1632%40paquier.xyz))
- **What is different this time:** P9 dropped per-file fsyncs in timeline-switch cleanup, where renames were mixed in, with no guard. 0001 batches only unlinks and leaves timeline-switch cleanup unbatched; 0002 batches renames but adds a write-path fallback fsync.


## 3. Objections a reviewer is likely to raise

### 1. Gain is shown only as background checkpoint cleanup time

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer merges the recycle batching on the strength of a 139 ms to 27 ms cleanup saving. The checkpointer does this work in the background, so most users see no change in TPS or commit latency. The project keeps the added code with no measured user benefit.

Every number in messages #1, #2 and #3 measures checkpoint cleanup time. Message #3 says it is "not a TPS result". The only demand in the material is a second-hand note in message #1 that an earlier thread called these fsyncs a bottleneck. No field report, no -general or -bugs case, and no user stall caused by this cleanup appears.

Thread: [#1 (Ayush Tiwari, 2026-08-02)](https://www.postgresql.org/message-id/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), [#2 (Ayush Tiwari, 2026-08-18)](https://www.postgresql.org/message-id/CAJTYsWW5nRTuPkU%2BmwM2ohN3L4SxN4xwca-5a7s99dOr49Roug%40mail.gmail.com), [#3 (Ayush Tiwari, 2026-09-02)](https://www.postgresql.org/message-id/CAJTYsWU0ozR-SA3WJQK1eMR%2B65rv3x98KYYi29V9xQ0HKkzFFg%40mail.gmail.com)

**Question:** Which workload shows a user-visible stall, such as commit latency, WAL segment shortage or recovery time, caused by per-segment pg_wal fsyncs, and how large is it?

Counter-check: held. Message #1 cites an earlier thread that called these fsyncs a bottleneck, but that thread's content is not in the material. No message shows a user-visible effect, and message #3 says the numbers are not a TPS result.

### 2. Write-path complexity paid by everyone for checkpointer-only time savings

fixable · level 1b, rubric item 4 · desirability, performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A user on default settings upgrades and sees no TPS or latency change, because recycling already runs in the background. The project still carries the frontier logic on the WAL write path and must keep it correct in every later WAL change.

All numbers measure checkpoint cleanup time, and message #3 says "This is not a TPS result". The large gains appear with 1 MiB segments or 128 removed segments, which are unusual setups. 0001 gets part of the benefit without touching the write path. 0002's extra cost lands on shared code that all users run.

Thread: [#1 (Ayush Tiwari, 2026-08-02)](https://www.postgresql.org/message-id/CAJTYsWUy7ycUswwHeWdbQpKboRj9Cd1rfm2TwcV5WC_1T28pSw%40mail.gmail.com), [#2 (Ayush Tiwari, 2026-08-18)](https://www.postgresql.org/message-id/CAJTYsWW5nRTuPkU%2BmwM2ohN3L4SxN4xwca-5a7s99dOr49Roug%40mail.gmail.com), [#3 (Ayush Tiwari, 2026-09-02)](https://www.postgresql.org/message-id/CAJTYsWU0ozR-SA3WJQK1eMR%2B65rv3x98KYYi29V9xQ0HKkzFFg%40mail.gmail.com)

**Question:** Which workload shows lower commit latency or higher throughput from 0002 beyond what 0001 already gives?

Counter-check: held. The 44-segment case uses 16 MiB segments and ordinary settings, so the gain is not limited to unusual setups. Still, every number is cleanup time, and the author in message #3 doubts that 0002's plumbing is worth it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Both patches change when pg_wal directory entries for removed and renamed segments become durable. |
| WAL / replication / standby | **yes** | 0002 changes how recycled WAL segments are installed and adds a timeline-qualified frontier in XLogCtl. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds shared state that the checkpointer, every segment installer and XLogFileInit() must keep consistent. |
| performance on the hot path | **yes** | 0002 adds a frontier and batch-marker check, and a possible file and pg_wal fsync, to XLogFileInit(). |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Gain is shown only as background checkpoint cleanup time |

## 5. Questions the author must answer before anyone reviews code

1. Which workload shows a user-visible effect, such as commit latency, throughput or recovery time, from per-segment pg_wal fsyncs, and how large is it?
2. What does 0002 add in commit latency or throughput beyond what 0001 alone gives on the same workload?
3. Can the safety rule for recycled segments live in one function, so that a future segment installer cannot bypass it without failing a test?
4. Which processes can create or write a WAL segment during a restartpoint, archive recovery or promotion, and how does each one reach the frontier check?
5. After a failed batch fsync, why is a later pg_wal fsync trusted, and why does the marker stay set until restart?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows WAL segment installation and crash durability in xlog.c. Message #1 cc's two people from the earlier thread, but nobody other than the author has replied.
- **Liability:** For 0002, the owner signs for crash safety of commits written into recycled segments, on primaries and standbys, for as long as the frontier exists. For 0001, the owner signs only for obsolete files that may reappear after a crash.
- **Author follow-up:** The author posted four versions in three months, rebased twice, and reported and fixed a hole in the v2 invariant in message #3.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: This applies to 0002 only. A benchmark or field report where per-segment pg_wal fsyncs cause a measurable commit-latency, throughput or recovery-time problem, and where 0002 removes it beyond what 0001 does, would change the answer.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** 0001 is small, limited to unlinks, and drew no objection, so it can go ahead by itself. 0002 is marked WIP, puts a new invariant on the WAL write path, and the author asks in messages #1 and #3 for a simpler one, while no user-visible gain is shown.

## Appendix

### Held up in the counter-check, not listed above

- `replication-ondisk-1` (fixable, level 2) Recycle durability frontier not shown to cover standby and recovery paths. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-2` (fixable, level 2) After a failed batch fsync, later fsyncs are trusted to finish the renames. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) 0002 spreads one durability invariant across checkpoint cleanup and WAL writes. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Failed recycle batch moves directory fsyncs onto the WAL write path until restart. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Batched recycle renames reopen the crash hazard that stopped an earlier attempt. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same bypass risk and asks a more concrete question. [P9] deferred the fsync with no guard, so it supports the hazard but not a judgment on the frontier design.
- `demand-2` 0002 puts write-path cost on everyone for a narrow group. Refuted (not specific): Duplicate of maintenance-fit-3 for the question of who pays and who gains, and of maintenance-fit-2 for the cost after a failed batch. A shared-memory check once per new segment is also not a measurable cost for other users.

### Review angles

- prior-attempts: 1 objection(s). No prior-attempt objection against 0001: it batches only unlinks, and a lost unlink only leaves an obsolete file that a later checkpoint removes. The pre-allocating WAL files thread (message #1, [P15]) is where the idea came from; its withdrawal reason is not recorded.
- standards-compat: 0 objection(s). The proposal changes no user-facing contract: no SQL syntax, GUC, catalog object, pg_dump output, psql behaviour or client-visible change; the only new surface is a test injection point. Existing queries, scripts and dumps behave the same after upgrade. The one visible difference, obsolete WAL files reappearing in pg_wal after a crash in 0001, is removed by a later checkpoint and does not belong to this angle.
- replication-ondisk: 2 objection(s). 0001 deletes only segments already past the checkpoint and archiving, so files that come back after a crash look harmless, and I have no objection to it from this angle. The objections cover 0002.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes when the checkpointer fsyncs pg_wal while removing and recycling WAL segments, adds no SQL surface, GUC, role, view or setting, and gives unprivileged users no new action, data or timing signal; the only additions are internal injection points and a TAP test. Crash durability concerns belong to other angles.
- maintenance-fit: 3 objection(s). 0001 is small, separable and keeps the existing LOG-level policy. I have no maintenance objection to it alone.
- already-exists: 0 objection(s). No existing feature, option or extension does this job. A larger --wal-segsize (initdb or pg_resetwal) cuts the number of segments, and the author's numbers show the cost dropping with it, but it is a cluster-wide layout change and does not remove the per-segment fsyncs. The patch changes internal checkpointer behaviour that no extension or hook can reach, so the question of why it belongs in core does not apply.
- demand: 2 objection(s). 0001 is small, adds no write-path cost, and has measured gains when many segments are removed. No demand objection is raised against 0001 on its own.

### Precedents retrieved

- P1: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P2: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P3: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P4: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P5: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P6: [Fix calculations on WAL recycling](https://commitfest.postgresql.org/patch/1730/) (cf, Rejected, 2018-07-23) - judged not relevant; reason summarised from its thread
- P7: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant; reason summarised from its thread
- P8: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Speed up WAL file removal during recovery](https://commitfest.postgresql.org/patch/1530/) (cf, Returned with feedback, 2018-09-30) - used in the review; reason summarised from its thread
- P10: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3668/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P11: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P12: [option to disable WAL recycling](https://commitfest.postgresql.org/patch/1700/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P14: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P15: [pre-allocating WAL files](https://commitfest.postgresql.org/patch/3158/) (cf, Withdrawn, 2022-07-25) - used in the review
- P16: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P17: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P18: [reduce impact of lengthy startup and checkpoint tasks](https://commitfest.postgresql.org/patch/3448/) (cf, Returned with feedback, 2023-09-01) - used in the review
- P19: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P20: [Improvements for pg_wal_replay_wait()](https://commitfest.postgresql.org/patch/5167/) (cf, Returned with feedback, 2024-12-10) - judged not relevant
- P21: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant
- P22: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - judged not relevant
- P23: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant
- P24: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 162 s; cost reported by the CLI: $1.57
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:18:55.118Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
