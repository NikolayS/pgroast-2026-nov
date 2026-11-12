# Design pre-review: Reset unlogged relations before syncing the data directory?

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7406](https://commitfest.postgresql.org/patch/7406/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [Reset unlogged relations before syncing the data directory?](https://www.postgresql.org/message-id/flat/CAJTYsWW_tx5ZacHLjCLHyvPkcjc%3Dy3OncLJJ4xyL7rwOFzGkkw%40mail.gmail.com), started 2026-10-07
- Messages read: 4 of 4
- Patch prose read: v2-0001-Reset-unlogged-relations-before-syncing-the-data-directory.patch, v1-0001-Reset-unlogged-relations-before-syncing-the-data-directory.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

The goal is clear and the cost falls on nobody who does not benefit, so levels 1a and 1b pass. One fixable level 2 objection survived, replication-ondisk-1: SyncDataDirectory() now runs after InitWalRecovery(), and the thread has no answer on why that is safe. The author asks this in message #1 and no committer has replied. A committer who knows the startup code has to confirm the ordering.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the exact position of SyncDataDirectory() relative to the steps inside InitWalRecovery() could not be checked. Message #4 was not shown.

## The proposal, restated

**Problem.** At crash startup, SyncDataDirectory() fsyncs unlogged relation forks that the startup process then removes. That writeback is wasted work and slows crash restart for clusters with large unlogged tables.

**Proposed design.** On crash starts, the patch runs the unlogged relation cleanup after InitWalRecovery() has restored tablespace links and before SyncDataDirectory(). Init forks and surviving files are still synced before replay, and the end-of-recovery checkpoint does not move. Recovery after a clean shutdown keeps the later cleanup, which runs after pg_control records recovery.

**What the author claims.**
- With a 1.14 GiB unlogged table, crash startup dropped from a 26.34 s median to 0.40 s with recovery_init_sync_method=fsync.
- With recovery_init_sync_method=syncfs, the median dropped from 22.58 s to 0.30 s, because syncfs cannot skip individual files.
- The author says the gains may come from dirty data left in the OS page cache after pg_ctl stop -m immediate.
- Keeping the late cleanup after a clean shutdown stops a failed start from deleting unlogged files while pg_control still allows a restart without recovery.

**Thread state.** Chao Li agreed with the change, asked for an explicit InRecovery check, and gave v2 an LGTM. No committer has replied, and the author asked whether the initial sync must happen before InitWalRecovery().

## 2. Prior attempts

### speed up pg_rewind by using copy_file_range() and skipping some fsync (related)

- **Link:** [https://commitfest.postgresql.org/patch/2990/](https://commitfest.postgresql.org/patch/2990/)
- **Thread:** [https://www.postgresql.org/message-id/flat/7C1703E7-F3F3-43FA-86EB-177C671BF33C@vmware.com](https://www.postgresql.org/message-id/flat/7C1703E7-F3F3-43FA-86EB-177C671BF33C@vmware.com)
- **Date:** decided 2021-10-01; started 2021-01-27
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier asked for pg_rewind benchmarks: many small files for the fsync patch, and large and small file sets on XFS with reflink for copy_file_range(). He also asked to drop the copy_file_range_support flag, because one EXDEV failure would turn copy_file_range() off for all other tablespaces and mount points, and to fix double-counted progress on fallback. The fsync patch skips files such as configuration files included from postgresql.conf. The visible messages do not show a reply after his last review.
- **Reason (quote):** "Anyway, I think that it would be good to do more benchmarking for this patch first, and the patch you are proposing is enough for that." ([message](https://www.postgresql.org/message-id/YR88OzOxILlDtLVd%40paquier.xyz))
- **What is different this time:** P4 skipped fsync of unchanged files in pg_rewind, a different tool and code path. This patch reorders unlogged cleanup and SyncDataDirectory() at crash startup. P4's request for more benchmarks is relevant because message #1 measured only without a cache flush.


## 3. Objections a reviewer is likely to raise

### 1. InitWalRecovery() now runs before the data directory is synced

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

The server crashes and restarts. InitWalRecovery() reads the checkpoint record and backup_label from the OS page cache, restores tablespace links and durably renames tablespace_map. Power is lost before SyncDataDirectory() finishes, so the next start finds the renamed map but may not find links or WAL that were never flushed.

The cleanup can only run after the tablespace links exist, so the patch has to move SyncDataDirectory() after InitWalRecovery(). Today the sync runs first, so recovery setup only reads state that is already durable. The proposal does not list which reads and durable renames in InitWalRecovery() would now depend on unsynced files. In message #1 the author asks this same question, and no committer has answered it in messages #2-#4.

Thread: [#1 (Ayush Tiwari, 2026-10-07)](https://www.postgresql.org/message-id/CAJTYsWW_tx5ZacHLjCLHyvPkcjc%3Dy3OncLJJ4xyL7rwOFzGkkw%40mail.gmail.com), [#2 (Chao Li, 2026-10-10)](https://www.postgresql.org/message-id/E22A0BD3-432F-4CF6-94FB-56717CA3E8FA%40gmail.com), [#4 (Chao Li, 2026-10-10)](https://www.postgresql.org/message-id/766EEB3A-4F42-4331-866A-4A38277522B6%40gmail.com)

**Question:** Which files does InitWalRecovery() read or rename on a crash start, and why is each step safe if the OS loses unsynced data before SyncDataDirectory() completes?

Counter-check: held. The sync does move after InitWalRecovery(), and message #1 asks whether that is safe without getting an answer. The stated scenario is weaker than written, because tablespace links are created after the sync today too, but the question about reads of unsynced files stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch changes when unlogged relation files are removed relative to the crash-start data directory sync; no format change. |
| WAL / replication / standby | **yes** | The patch reorders InitWalRecovery(), unlogged cleanup and SyncDataDirectory() in the startup process in xlog.c. |
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
| maintenance burden ("who maintains this in 10 years") | **yes** | The unlogged cleanup now runs at two points chosen by start type, each with its own ordering reason. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which files does InitWalRecovery() read or rename on a crash start, and why is each step safe if the OS loses unsynced data before SyncDataDirectory() completes?
2. What happens if the server crashes or loses power after the early cleanup but before SyncDataDirectory() finishes?
3. What are the patched and unpatched crash startup times when the OS page cache is flushed before restart?
4. Does a test cover a crash start with unlogged tables in a tablespace under the new ordering?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows StartupXLOG, InitWalRecovery() and the crash-start fsync rules has to sign off; no committer has replied in the visible thread.
- **Liability:** The owner signs for crash-start durability under the new ordering, including a second crash before the sync finishes, for as long as the ordering stands.
- **Author follow-up:** The author posted v2 three days after the review in message #2 and asked an open design question, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 4 KiB in one file and helps clusters with large unlogged tables on crash restart, with no user-visible surface. It waits on a committer who knows the startup code to answer the ordering question from message #1.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Split cleanup ordering relies on untested startup invariants. Refuted (no failure scenario): The failure needs a later patch that reorders StartupXLOG against a reason the commit message already states, and v2 adds the explicit InRecovery check. The request for a TAP test is level 4, which is out of scope here.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent tried this reordering, and P10's reason, about a custodian process and file-name wraparound, does not apply. P4 asked for benchmarks, but here that is a performance-evidence question for the other angles, not a prior rejection reason. The thread points to no earlier discussion of the same idea.
- standards-compat: 0 objection(s). The patch changes only the order of startup steps in xlog.c, with no new syntax, GUC, catalog object, psql change or dump content. Unlogged tables are already emptied after a crash, so users and clients see the same result. Crash-safety of the new ordering belongs to another angle.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL record, catalog, protocol or ABI. Unlinks done before the sync become durable once SyncDataDirectory() fsyncs the directories. A crash during the early cleanup leads to another crash start, which runs the cleanup again.
- security: 0 objection(s). The patch only reorders crash-startup steps inside the startup process, which runs before any client can connect. It adds no SQL surface, GUC, privilege, role or monitoring output, so no user can reach it or learn anything new. Its durability and crash-ordering questions belong to the safety angle.
- maintenance-fit: 1 objection(s). The change is small, confined to crash startup, and adds no per-query, per-tuple or per-backend cost, no new GUC and no external dependency. No committer has engaged yet, which affects committability, not the merits.
- already-exists: 0 objection(s). No existing feature does this job. Both values of recovery_init_sync_method, fsync and syncfs, still write back the unlogged forks, and fsync = off gives up durability for every file. No extension or hook can change the order of steps inside the startup process, so the change has to be in core. The patch reorders existing calls and adds no second mechanism. Not checked: whether the reorder is safe and whether the gain holds with a cold page cache; other angles cover these.
- demand: 0 objection(s). No demand objection stands. The patch adds no GUC, syntax or catalog change and touches only xlog.c, so users who do not benefit pay almost nothing. The gain applies whenever the postmaster does a crash restart without an OS reboot, which is the default restart_after_crash case where dirty unlogged pages remain in the page cache. The thread has no field report, and the benchmark covers only the dirty-cache case, but a benefit-only internal change does not need broad demand evidence.

### Precedents retrieved

- P1: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P2: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant; reason summarised from its thread
- P3: [Detectable crashes and unlogged table resets](https://commitfest.postgresql.org/patch/3183/) (cf, Returned with feedback, 2022-03-07) - judged not relevant; reason summarised from its thread
- P4: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - used in the review; reason summarised from its thread
- P5: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant; reason summarised from its thread
- P6: [Startup Centralization](https://commitfest.postgresql.org/patch/2498/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P7: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P8: [Generalize ereport_startup_progress infrastructure](https://commitfest.postgresql.org/patch/3807/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P9: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - judged not relevant
- P10: [reduce impact of lengthy startup and checkpoint tasks](https://commitfest.postgresql.org/patch/3448/) (cf, Returned with feedback, 2023-09-01) - used in the review; reason summarised from its thread
- P11: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P12: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P13: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P14: [Inconsistency in startup process's MyBackendId and procsignal array registration with ProcSignalInit()](https://commitfest.postgresql.org/patch/3357/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P15: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P16: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - judged not relevant
- P17: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant
- P19: [Speed up WAL file removal during recovery](https://commitfest.postgresql.org/patch/1530/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P20: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant
- P21: [Problem in 'ORDER BY' of a column using a created collation?](https://commitfest.postgresql.org/patch/5990/) (cf, Returned with feedback, 2025-10-21) - judged not relevant
- P22: [logical column ordering (WIP)](https://commitfest.postgresql.org/patch/21/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P23: [Make the optimiser aware of partitions ordering](https://commitfest.postgresql.org/patch/1093/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P24: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 127 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:51:38.745Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
