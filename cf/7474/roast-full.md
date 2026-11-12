# Design pre-review: Small gaps in IoWorkerMain() error handling

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7474](https://commitfest.postgresql.org/patch/7474/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [\[PATCH\] Small gaps in IoWorkerMain() error handling](https://www.postgresql.org/message-id/flat/CAJTYsWVQ%3DWS%2BiCaKtWpB_Yv7wnDRF-SyLwCN3XrJP08mH%3DKJWQ%40mail.gmail.com), started 2026-10-16
- Messages read: 1 of 1
- Patch prose read: v1-0001-aio-Set-error_errno-before-error_ioh-in-I-O-workers.patch, v1-0002-aio-Hold-interrupts-earlier-in-I-O-workers.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived the panel at levels 1a, 1b or 2. The problem is stated with a reproduction path, the two fixes are local to IoWorkerMain(), and they add no user-visible surface. Confidence is low because only message #1 exists, nobody has replied, and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that the DEBUG4 message is the only interrupt-processing point before HOLD_INTERRUPTS() is unverified. Whether setting error_errno earlier changes any other error path also could not be judged.

## The proposal, restated

**Problem.** In IoWorkerMain(), an error raised while emitting the DEBUG4 "processing IO" message leaves error_errno at 0, which trips Assert(error_errno != 0) or gives a misleading "read only 0 of N bytes" error. A SIGINT handled during the same message makes the worker exit with a dequeued IO never completed, so waiting backends hang. Both cases need DEBUG4 logging from I/O workers.

**Proposed design.** Patch 0001 sets error_errno (ENOENT, as the reopen failure path already uses) before error_ioh. Patch 0002 moves the DEBUG4 message below HOLD_INTERRUPTS(), so errfinish() cannot process a die interrupt in that window. Both patches target back branches through 18.

**What the author claims.**
- The author hit the assertion by injecting errors from an emit_log_hook, and says ordinary users probably do not hit it.
- Without assertions, a read fails with a misleading "read only 0 of N bytes" error.
- The DEBUG4 message is the only place that processes interrupts before HOLD_INTERRUPTS().
- Moving the message is less invasive than failing owned IOs from an exit callback, which suits backpatching.

**Thread state.** Only the first message is visible. Nobody has replied, and the CommitFest entry is in Needs review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-16; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Both commit messages say "Backpatch-through: 18", so the change lands in a stable branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Patch 0002 relies on no later code adding interrupt processing between dequeue and HOLD_INTERRUPTS() in method_worker.c. |
| performance on the hot path | **yes** | Both patches reorder statements in the per-IO loop of IoWorkerMain(); the panel found no added cost. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can any code between the dequeue and HOLD_INTERRUPTS(), other than the DEBUG4 message, raise an error or process interrupts?
2. Which errno should the handler report when the failure is unrelated to the file, given that message #1 calls ENOENT a poor fit?
3. Should master get the exit callback that fails owned IOs, with the smaller reordering used only for the back branch?
4. Can an injection point or a test module reproduce both failures without a hand-made emit_log_hook?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the AIO worker method and the interrupt handling of auxiliary processes would have to own it. Nobody has replied in the visible thread.
- **Liability:** The owner signs for a change in a stable branch to the I/O worker error and interrupt path. A mistake there could cause wrong IO completion or hangs, and the reordering must keep holding as code is added later.
- **Author follow-up:** No sign either way. Message #1 leaves the errno choice and the exit-callback alternative open, which suggests the author expects to revise.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Both failures need DEBUG4 logging from I/O workers, so few installations can hit them, but one of them hangs backends. The two patches are under 2 KiB each, touch one file and add no user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The one listed row [P5] shares only the topic of interrupt handling. It is about a standby backend blocked on a client write, and its recorded reason (interrupts held while sending an error message) does not apply to moving an I/O worker DEBUG4 message below HOLD_INTERRUPTS(). Message #1 does not point to any earlier discussion of this fix.
- standards-compat: 0 objection(s). The patches change no SQL syntax, GUC, catalog, pg_dump output, psql behaviour or client-visible contract. They only reorder an errno assignment and a DEBUG4 message inside the I/O worker loop. The one visible effect is the errno reported in a rare error path, ENOENT, which the reopen failure path already uses.
- replication-ondisk: 0 objection(s). The patches change only control flow inside IoWorkerMain(): the order of an errno assignment and of HOLD_INTERRUPTS(). They add no on-disk format, WAL record, catalog, wire protocol or exported API change, so backpatching to 18 has no ABI or pg_upgrade concern from this angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Both paths need DEBUG4 logging from I/O workers, a SIGINT to an auxiliary process, or an emit_log_hook, and an unprivileged user can arrange none of these. The patches add no new access, data exposure or default.
- maintenance-fit: 0 objection(s). Two small, local bugfixes in one file. They add no GUC, API, catalog or hot-path cost, and the code only runs when I/O workers log at DEBUG4. The exit-callback alternative in message #1 would also protect future code added between dequeue and HOLD_INTERRUPTS(); a committer might want it in master only, but that does not argue against these back-branch fixes. [P5] concerns a different interrupt path and does not apply.
- already-exists: 0 objection(s). This is a bug fix in IoWorkerMain() error handling. No existing feature, option or extension covers it, and an extension cannot change how the I/O worker orders its error state and interrupt holding. The only other approach discussed is the author's own exit-callback alternative, which is a design choice inside core and falls outside this angle.
- demand: 0 objection(s). This is a bug fix for an assertion failure and a possible backend hang, so the demand test does not apply in the usual way. The trigger is rare and needs DEBUG4 logging from I/O workers, but the fix is small, the cost falls on nobody, and committers do not usually ask for field reports before fixing a hang.

### Precedents retrieved

- P1: [s/shm_mq_iovec/struct iovec/](https://commitfest.postgresql.org/patch/4928/) (cf, Returned with feedback, 2024-07-15) - judged not relevant; reason summarised from its thread
- P2: [Test for function error in logrep worker](https://commitfest.postgresql.org/patch/4046/) (cf, Withdrawn, 2023-04-05) - judged not relevant; reason summarised from its thread
- P3: [Asynchronous and "direct" IO support for PostgreSQL](https://commitfest.postgresql.org/patch/3316/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P4: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant; reason summarised from its thread
- P5: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - used in the review; reason summarised from its thread
- P6: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P7: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P8: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Pruning never visible changes](https://commitfest.postgresql.org/patch/3900/) (cf, Returned with feedback, 2022-10-27) - judged not relevant; reason summarised from its thread
- P10: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant
- P11: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant
- P12: [Revert setting client_min_messages to 'debug1' in new tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5655565c077c53b6e9b4b9bfcdf96439cf3af065) (revert, Reverted, 2019-03-13) - judged not relevant
- P13: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P14: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [Revert 019_replslot_limit.pl related debugging aids.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3f8148c256e067dc2e8929ed174671ba7dc3339c) (revert, Reverted, 2022-07-05) - judged not relevant
- P16: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant
- P17: [Improvements of worker_spi.c example](https://commitfest.postgresql.org/patch/4315/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P18: [add assertion for palloc in signal handlers](https://commitfest.postgresql.org/patch/6498/) (cf, Withdrawn, 2026-08-24) - judged not relevant
- P19: [Asynchronous execution of ForeignScans](https://commitfest.postgresql.org/patch/751/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P20: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P21: [Undo worker and transaction rollback](https://commitfest.postgresql.org/patch/1828/) (cf, Withdrawn, 2019-09-09) - judged not relevant
- P22: [Add description about using geqo to EXPLAIN or debug log.](https://commitfest.postgresql.org/patch/3680/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [asynchronous execution support for Custom Scan](https://commitfest.postgresql.org/patch/3813/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 46 s; cost reported by the CLI: $0.47
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:59:35.214Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
