# Design pre-review: A bug in recovery prefetcher, triggering asserts / FATAL: checkpoint request failed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7511](https://commitfest.postgresql.org/patch/7511/) (Needs review); authors: Tomas Vondra (fuzzycz)
- Thread: [A bug in recovery prefetcher, triggering asserts / FATAL: checkpoint request failed](https://www.postgresql.org/message-id/flat/a797f236-286a-43e6-9b82-5f3d70bd5e99%40vondra.me), started 2026-10-25
- Messages read: 4 of 4
- Patch prose read: 0001-Forget-the-prefetcher-s-current-record-in-XLogPrefet.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The thread states the bug, its two failure modes and the fix, and three people confirmed the fix in messages #2 to #4. The patch clears one field of the prefetcher's private state and adds no user-visible surface. Confidence is medium because the code was not read and the claim about the other callers rests on the commit message.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that the other callers of XLogPrefetcherBeginRead never have a current record is unchecked. The stability of the TAP test across branches and buildfarm animals could not be judged.

## The proposal, restated

**Problem.** XLogPrefetcherBeginRead resets the prefetcher queue but keeps prefetcher->record, so FinishWalRecovery can fail to re-read the last replayed record when recovery stops at a recovery target. Assert builds then crash, and other builds write a wrong timeline switch point and fail with "FATAL: checkpoint request failed". The bug affects Postgres 15 and later, mostly with maintenance_io_concurrency set to 7 or less.

**Proposed design.** The patch sets prefetcher->record to NULL in XLogPrefetcherBeginRead, so the next read decodes the record at the new position. A new TAP test stops recovery just before a multi-block XLOG_FPI record from an SP-GiST index build, with maintenance_io_concurrency = 1. The fix is backpatched through 15.

**What the author claims.**
- The bug has existed since Postgres 15.
- On non-assert builds the new timeline history file gets a wrong switch point, and a restart replays WAL past the recovery target.
- It triggers when maintenance_io_concurrency is 7 or less and the recovery target is shortly before a bulk XLOG_FPI record.
- The other callers of XLogPrefetcherBeginRead run before the redo loop, so they never have a current record.
- The TAP test works unchanged in back branches 15 to 18.

**Thread state.** Chao Li, shihao zhong and Surya Poondla tested or reviewed the fix and agree with it. Surya suggests a separate patch that makes FinishWalRecovery PANIC if the last record cannot be re-read; whether to keep the TAP test is still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The fix changes xlogprefetcher.c, which runs in the startup process and decides where WAL ends at a recovery target. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a backpatch through 15, with a TAP test written to run unchanged in 15 to 18. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a new recovery TAP test, 061_recovery_target_prefetch.pl, that depends on the WAL pattern of an SP-GiST index build. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the bug occur with the default maintenance_io_concurrency, for example with a record that has enough block references that need I/O?
2. Should the TAP test be committed and backpatched, or kept only as a reproducer?
3. How can a user detect and repair a cluster whose timeline history file already has a wrong switch point from this bug?
4. Should a separate patch make FinishWalRecovery fail loudly when it cannot re-read the last replayed record?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The CommitFest entry lists Tomas Vondra as author and the commit message carries a backpatch line; the owner needs to know the WAL replay and recovery prefetcher code.
- **Liability:** A change to recovery-end behaviour in four or more back branches, plus a TAP test that must stay stable on the buildfarm in each of them.
- **Author follow-up:** The author found the bug while working on other recovery prefetcher improvements, which suggests continued work in this area; the thread has no explicit statement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug needs a recovery target shortly before a bulk XLOG_FPI record and mostly maintenance_io_concurrency of 7 or less, but the result is a wrong timeline switch point and replay past the target. The fix is one pointer reset in one file, and three people confirmed it.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] is the 2021 revert of the whole recovery prefetching feature, made for release-cycle timing. Its reason does not apply to a one-line bug fix in the later, committed prefetcher. The thread cites no earlier discussion of this bug, and the three replies (messages #2-#4) agree with the fix.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, psql behaviour or dump content, so it changes no user-facing contract; it only corrects recovery-end behaviour, and the backpatch through 15 fits normal bugfix practice. The open questions are outside this angle: whether to commit the TAP test, and how users can find clusters whose timeline history file got a wrong switch point.
- replication-ondisk: 0 objection(s). The fix changes no on-disk format, WAL record, catalog, protocol or exported struct. It clears one field in the prefetcher's private state, so backpatching to 15 creates no ABI or pg_upgrade problem. Release notes could tell users how to spot timeline history files with a wrong switch point that the bug has already written, but that does not argue against the fix. [P3] concerns the original feature's revert and does not apply to this bug fix.
- security: 0 objection(s). The patch only clears a stale internal pointer in the recovery prefetcher. It adds no privilege, role, setting, view, error text or user-reachable path, and only the startup process runs this code. Recovery targets and maintenance_io_concurrency already require superuser or file access, so there is nothing to object to from a security angle.
- maintenance-fit: 0 objection(s). This is a backpatched bugfix: one pointer reset, in the module that already owns that state. It adds no GUC, catalog change, API or hot-path work, and the cost of skipping it is wrong timeline history files and replay past the recovery target. [P3] reverted the original feature in 2021 and does not argue against fixing it now; whether to keep the TAP test is a test-level question and out of scope here.
- already-exists: 0 objection(s). This is a bug fix in recovery prefetcher state handling, so no existing feature, extension or setting meets the need. Setting recovery_prefetch = off avoids the bug but leaves it in place, so it is not an alternative. [P3] is about the original feature being reverted and does not apply.
- demand: 0 objection(s). This is a backpatched bug fix for a crash and a wrong timeline switch point in point-in-time recovery, which can then replay WAL past the recovery target, so the demand angle does not apply. Three people confirmed the bug and the fix (messages #2-#4); the open questions about the TAP test and the PANIC check are not demand concerns.

### Precedents retrieved

- P1: [Be strict when request to flush past end of WAL in WaitXLogInsertionsToFinish](https://commitfest.postgresql.org/patch/7294/) (cf, Withdrawn, 2026-09-12) - judged not relevant; reason summarised from its thread
- P2: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P3: [Revert recovery prefetching feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c2dc19342e05e081dc13b296787baa38352681ef) (revert, Reverted, 2021-05-10) - used in the review
- P4: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3690/) (cf, Withdrawn, 2022-06-17) - judged not relevant; reason summarised from its thread
- P5: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P6: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant; reason summarised from its thread
- P7: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P8: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P9: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - judged not relevant; reason summarised from its thread
- P10: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P11: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant
- P12: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant
- P13: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant
- P14: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P15: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P16: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [Flush some statistics within running transactions](https://commitfest.postgresql.org/patch/6389/) (cf, Returned with feedback, 2026-03-25) - judged not relevant
- P18: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant
- P19: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant
- P20: [pg_stat_database update stats_reset only by pg_stat_reset](https://commitfest.postgresql.org/patch/2116/) (cf, Withdrawn, 2019-07-19) - judged not relevant
- P21: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P22: [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (cf, Withdrawn, 2024-01-31) - judged not relevant
- P23: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P24: [Allow tuple visibility checks without hint-bit maintenance](https://commitfest.postgresql.org/patch/7212/) (cf, Withdrawn, 2026-08-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 52 s; cost reported by the CLI: $0.60
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:02.896Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
