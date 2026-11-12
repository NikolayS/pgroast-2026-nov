# Design pre-review: Fix recovery conflict canceling the startup progress timeout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7521](https://commitfest.postgresql.org/patch/7521/) (Needs review); authors: Aidar Imamov (gipergeek)
- Thread: [Fix recovery conflict canceling the startup progress timeout](https://www.postgresql.org/message-id/flat/1D46953B-2FE9-4611-9B46-FAE91184AE99%40gmail.com), started 2026-10-26
- Messages read: 3 of 3
- Patch prose read: v1-Fix-recovery-conflict-canceling-the-startup-progress.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated, reproduced on 15 through master by the reviewer, and the fix restores behaviour that log_startup_progress_interval already promises. The one surviving objection, prior-attempts-1, is a fixable design point. It asks how the lock and buffer-pin waits behave now that the progress timeout stays armed and can wake the startup process. A committer who knows the recovery-conflict code can settle it, and the backpatch to 15 needs the same person.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check whether the resolvers re-arm STANDBY_DEADLOCK_TIMEOUT on each wakeup, whether the outdated comment is updated, or whether other startup-process timeouts relied on the old cleanup.

## The proposal, restated

**Problem.** In archive recovery with hot_standby=on, one lock or buffer-pin recovery conflict stops "redo in progress" log messages for the rest of redo. Redo still finishes, but an admin watching a long PITR sees progress stop and may think recovery is stuck. The bug exists since 15, when STARTUP_PROGRESS_TIMEOUT was added.

**Proposed design.** ResolveRecoveryConflictWithLock() and ResolveRecoveryConflictWithBufferPin() end with disable_all_timeouts(false), which also cancels STARTUP_PROGRESS_TIMEOUT. The patch makes each function disable only the two timeouts it enabled. The reviewer adds a TAP test in 031_recovery_conflict.pl with a gated Perl restore_command that holds replay around the conflicting record.

**User-visible surface.**
- "redo in progress" messages under log_startup_progress_interval keep appearing after a lock or buffer-pin recovery conflict.

**What the author claims.**
- One conflict cut progress messages from 22 to 7 in the author's reproduction; redo and data stay correct.
- Only lock and buffer-pin conflicts are affected, because other conflict types do not wait on timeouts.
- The fix is a backpatch candidate to 15; the reviewer reports v1 applies cleanly to 15 through master.
- The reviewer reproduced the bug and confirmed the fix on 15 through master, and the v3 test passes on CI including Windows.
- The v3 test adds about 3 seconds to 031 and does not depend on replay speed.

**Thread state.** Only the reviewer, Ritesh Kumar, replied; he supports the fix, posted v3 with a test, and suggests backpatching the fix to 15 and the test to 17. No committer has commented, and the author moved the entry back to Needs review without saying why in the visible thread.

## 2. Prior attempts

### Revert "Get rid of the dedicated latch for signaling the startup process". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7)
- **Thread:** [https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com), [https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com), [https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi](https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi)
- **Date:** decided 2020-12-17; started 2020-11-04
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert ac22929a26, as well as the followup fix 113d3591b8. Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin(). It caused, for example, SIGHUP signal handler or walreceiver process to wake that startup process up unnecessarily frequently. Additionally, add the comments about why that dedicated latch that the reverted patch tried to get rid of should not be removed. Thanks to Kyotaro Horiguchi for the discussion.
- **Reason (quote):** "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()."
- **What is different this time:** [P3] removed the startup process's dedicated latch. This patch changes which timeouts the resolvers cancel, but it leaves the progress timeout armed as a periodic wakeup source during the same buffer-pin wait.


## 3. Objections a reviewer is likely to raise

### 1. Progress timeout stays armed during conflict waits, which wake the startup process

fixable · level 2, rubric item 7 · WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

An admin sets log_startup_progress_interval below deadlock_timeout during PITR with hot_standby=on. Each progress tick sets the latch, so the startup process wakes from its buffer-pin or lock wait and calls the resolver again. If the resolver re-arms STANDBY_DEADLOCK_TIMEOUT on each call, the deadlock check may never fire, and the wait lasts until max_standby_archive_delay, or forever at -1.

[P3] was reverted because it broke the assumption that a startup process waiting on a buffer-pin conflict is woken only by unpin or by the timeout that ResolveRecoveryConflictWithBufferPin() enabled. With this patch, STARTUP_PROGRESS_TIMEOUT stays armed through every conflict wait, which is the kind of extra wakeup [P3] warns about. Today this happens only during the first conflict; the patch makes it happen in all of them. Messages #1-#3 do not discuss wakeups during the wait.

**Precedents.**
- [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (Reverted, 2020-12-17); thread: [#1](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com), [#2](https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com), [#3](https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi)

Thread: [#1 (Aidar Imamov, 2026-10-26)](https://www.postgresql.org/message-id/1D46953B-2FE9-4611-9B46-FAE91184AE99%40gmail.com), [#3 (Ritesh Kumar, 2026-11-08)](https://www.postgresql.org/message-id/z-F0YA2lTC4VlSp9RRxzwY9HDXFFyfxWoeWd7y_ITHcYiK6BWW0kgwMFTv18HL82L1_sy0kDr91fJwtKmkTJ1rCx6OMI6niU73yIuFLej2I%3D%40rtshkmr.com)

**Question:** How do the lock and buffer-pin waits behave when the progress timeout fires repeatedly, for example with log_startup_progress_interval shorter than deadlock_timeout?

Counter-check: held. Any timeout firing sets the process latch, and both resolvers wait on that latch and re-arm STANDBY_DEADLOCK_TIMEOUT on each call, so the scenario is plausible. It needs log_startup_progress_interval below deadlock_timeout, which is not the default, and the thread does not discuss it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes timeout cleanup in the startup process's lock and buffer-pin recovery-conflict resolvers in standby.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The thread proposes backpatching the fix to 15 and the test to 17, with per-branch test variants. |
| maintenance burden ("who maintains this in 10 years") | **yes** | CF 7372 reworks the same resolvers and must keep the narrower cleanup; the test needs different variants on back branches. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How do the lock and buffer-pin waits behave when the progress timeout fires repeatedly, for example with log_startup_progress_interval shorter than deadlock_timeout?
2. Does any other startup-process code arm a timeout and rely on the old disable_all_timeouts() call to clear it?
3. Does the patch update or remove the comment that says the startup process has no other outstanding timeouts?
4. If CF 7372 is committed first, which entry carries the narrower cleanup into its reworked resolvers?
5. Why did the entry move from Ready for Committer back to Needs review on 2026-11-03?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the startup process's recovery-conflict waits and the timeout and latch code in standby.c would have to own it.
- **Liability:** The owner signs for startup-process wait behaviour during lock and buffer-pin conflicts on six branches, 15 through master, and for a new TAP test with per-branch variants.
- **Author follow-up:** The author changed the CF status on 2026-11-03 but has not replied in the thread after message #1; the reviewer wrote and maintains the test.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix touches one file and narrows two cleanup calls, and the reviewer verified it on 15 through master with a test. It affects admins who run queries during archive recovery with hot_standby=on, and it needs one answer on wakeups during the wait.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 1 objection(s). No earlier attempt at this fix is in the precedent list. Message #3 points to CF 7372 and CF 6876 as related work in the same area, but neither is a rejected attempt at this fix.
- standards-compat: 0 objection(s). The fix adds no syntax, GUC, catalog object, psql behaviour or dump state. The only user-visible change is that "redo in progress" messages under log_startup_progress_interval keep appearing after a lock or buffer-pin conflict, which the setting already promises, so back branches change only by restoring the documented behaviour. [P3] concerns startup-process wakeups, which is outside this angle.
- replication-ondisk: 0 objection(s). The patch adds no persistent or cross-node state and changes no on-disk format, WAL record, catalog, wire protocol or exported API, so it is safe to backpatch from this angle. It narrows which in-process timeouts the startup process clears after a lock or buffer-pin conflict on a hot standby. [P3] concerns startup-process wakeups during conflict waits, which this patch does not change.
- security: 0 objection(s). The patch only changes which startup-process timeouts the lock and buffer-pin conflict resolvers disable, so it has no privilege or exposure aspect. It adds no new privilege, view, message content or user-controlled input. A read-only standby user could already trigger these conflicts, and after the fix they no longer silence progress logging.
- maintenance-fit: 0 objection(s). I have no maintenance or fit objection. The patch narrows cleanup in two functions to the timeouts they enable, adds no hot-path cost and fits normal timeout practice; the v3 test fails loudly if CF 7372 restores disable_all_timeouts(). Without the code I could not check whether the outdated comment is updated, or whether any other startup-process timeout relied on the old blanket cancel. [P3] concerns latch wakeups and does not apply.
- already-exists: 0 objection(s). This is a bug fix for a timeout that existing code cancels by mistake. No existing feature, setting or extension restores "redo in progress" messages after a lock or buffer-pin conflict, and CF 6876 covers standby-mode replay, not archive recovery. The fix narrows existing disable_timeout() use and adds no second mechanism.
- demand: 0 objection(s). No demand objection. This fixes a bug in an existing feature, log_startup_progress_interval, so it restores documented behaviour and needs no new users. The cost is a narrower timeout cleanup in two resolvers, with no new UI, GUC or hot-path work, and the reviewer reproduced the bug on 15 through master (messages #1 and #2).

### Precedents retrieved

- P1: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P2: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P3: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - used in the review
- P4: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant; reason summarised from its thread
- P5: [Microsecond-based timeouts](https://commitfest.postgresql.org/patch/4245/) (cf, Returned with feedback, 2023-07-26) - judged not relevant; reason summarised from its thread
- P6: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Generalize ereport_startup_progress infrastructure](https://commitfest.postgresql.org/patch/3807/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P8: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [This adds a new GUC parameter that automatically rolls back prepared transactions that have remained unresolved beyond the configured timeout. The cleanup is performed by the checkpointer process.](https://commitfest.postgresql.org/patch/6611/) (cf, Rejected, 2026-07-01) - judged not relevant; reason summarised from its thread
- P10: [Stmt timeout error can be sent after a CommandComplete](https://commitfest.postgresql.org/patch/6394/) (cf, Withdrawn, 2026-01-14) - judged not relevant; reason summarised from its thread
- P11: [meson: Adjust test timeout for Valgrind builds](https://commitfest.postgresql.org/patch/6646/) (cf, Withdrawn, 2026-06-10) - judged not relevant
- P12: [Document AccessExclusive lock behaviour on standbys](https://commitfest.postgresql.org/patch/5920/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P15: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P16: [logical streaming for large in-progress transactions](https://commitfest.postgresql.org/patch/1429/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P17: [WIP: Pgbench Serialization and deadlock errors](https://commitfest.postgresql.org/patch/1170/) (cf, Returned with feedback, 2017-09-28) - judged not relevant
- P18: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P19: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P20: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P21: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P22: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P23: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [using an end-of-recovery record in all cases](https://commitfest.postgresql.org/patch/3365/) (cf, Withdrawn, 2022-01-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 102 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:28.241Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
