# Design pre-review: Deadlock detector fails to activate on a hot standby replica

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7372](https://commitfest.postgresql.org/patch/7372/) (Needs review); authors: Vitaly Davydov (vitprof), Xuneng Zhou (dylanzhou)
- Thread: [Deadlock detector fails to activate on a hot standby replica](https://www.postgresql.org/message-id/flat/44c24dcf-5710-410f-b1b6-d10b315f3d51%40postgrespro.ru), started 2026-01-19
- Messages read: 43 of 43
- Patch prose read: v7-0001-Fix-recovery-conflict-wait-loop-for-buffer-pins-o.patch, 0001-Fix-deadlock-detector-activation-in-startup-process.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

Nothing that survived questions the goal or the core design: this is a fix for a reproduced recovery stall, and the wait-loop approach was settled in review. The one surviving objection, maintenance-fit-2, is fixable: the stable-branch patches stop at v6.1 and lack the lock-conflict fix and the timeouts_armed flag from v10. What remains is a committer's call on the backpatch shape and on the commit order against the overlapping patches.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm here that the new inner loops cannot miss an unpin or a lock release, or that v10 applies to the stable branches. The claim that the v10 tests exercise the fixed path is also unchecked.

## The proposal, restated

**Problem.** On a hot standby, the startup process can deadlock with a backend while waiting for a buffer pin, for example when replaying XLOG_HEAP2_PRUNE_* records. An unrelated SIGALRM or latch wakeup before deadlock_timeout makes the wait loop reset its timeouts, so the deadlock timeout never fires. With max_standby_streaming_delay = -1, recovery stalls forever; the author reports this on versions 15 and later.

**Proposed design.** ResolveRecoveryConflictWithBufferPin gets an inner wait loop that rechecks after each wakeup whether the buffer is ready for cleanup, and keeps waiting without tearing down active timeouts when no timeout fired. A helper re-registers the startup process as BM_PIN_COUNT_WAITER before each wait. v10 extends the same handling to ResolveRecoveryConflictWithLock and adds TAP tests that use pg_log_backend_memory_contexts to cause unrelated wakeups.

**User-visible surface.**
- New internal C functions in bufmgr.h, such as PinCountWaiterCheckReadyForCleanup and RegisterPinCountWaiter (names changed across versions).

**What the author claims.**
- Recovery can hang forever when deadlock_timeout is greater than log_startup_progress_interval, on versions 15 and later.
- The fix also avoids needless timeout teardown and re-arming on unrelated wakeups in the standby timeout path.
- Patches for REL_14_STABLE through REL_18_STABLE exist and are meant for backpatching.
- ResolveRecoveryConflictWithLock has the same bug, shown by a separate reproducer script.
- A reviewer reports that v10 cancels the deadlocked query after deadlock_timeout.

**Thread state.** Fujii Masao, Xuneng Zhou, JoongHyuk Shin and shihao zhong reviewed; v10 has a fix, TAP tests and an optional refactor commit, and the last review says it looks good. Open: the case where the standby delay has already passed (CF 6445), the overlap with other patches in this area, and a later refactor of the bufmgr.c and standby.c coupling.

## 2. Prior attempts

### Revert "Get rid of the dedicated latch for signaling the startup process". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7)
- **Thread:** [https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com), [https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com), [https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi](https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi)
- **Date:** decided 2020-12-17; started 2020-11-04
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert ac22929a26, as well as the followup fix 113d3591b8. Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin(). It caused, for example, SIGHUP signal handler or walreceiver process to wake that startup process up unnecessarily frequently. Additionally, add the comments about why that dedicated latch that the reverted patch tried to get rid of should not be removed. Thanks to Kyotaro Horiguchi for the discussion.
- **Reason (quote):** "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()."
- **What is different this time:** P1 was reverted because extra wakeups broke the buffer pin wait in ResolveRecoveryConflictWithBufferPin(). This proposal accepts that unrelated wakeups happen, as message #3 notes, and makes the wait loop recheck and keep waiting, so it no longer relies on that assumption.


## 3. Objections a reviewer is likely to raise

### 1. Backpatches lag the master fix and miss the lock case

fixable · level 2, rubric item 6 · backpatchability, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A committer pushes v10 to master and the existing v6.1 patches to REL_14 through REL_18. Stable branches keep the ResolveRecoveryConflictWithLock hang, and their buffer-pin loop differs from master's. Later fixes in this area, such as CF 6445 or the patch in message #24, then need separate versions for each branch.

Message #19 posted the stable-branch patches at v6.1. Later versions added the refcount recheck (messages #32-#34), the lock-conflict fix and the timeouts_armed flag (message #39), and none of these were backported. Other patches touching ResolveRecoveryConflictWithBufferPin are waiting on this one (messages #26, #31), and the bufmgr.c/standby.c refactor is deferred (message #29), so the branches will keep diverging.

Thread: [#19 (Vitaly Davydov, 2026-06-26)](https://www.postgresql.org/message-id/7b1e8353-3d0a-48a1-bac6-2ac6ca91fbde%40postgrespro.ru), [#24 (JoongHyuk Shin, 2026-08-04)](https://www.postgresql.org/message-id/CACSdjfMBDtxiFyA3eEVNJRQi1rufPGEjXOtAQ3qQkVc-P%3D187w%40mail.gmail.com), [#26 (Xuneng Zhou, 2026-08-05)](https://www.postgresql.org/message-id/CABPTF7WLTqtAL8DthdoMk5tKiVa22C4Urkbrcw78186eayracQ%40mail.gmail.com), [#29 (Vitaly Davydov, 2026-08-13)](https://www.postgresql.org/message-id/CAG5KqAbuFFgVfNoDHhiKGsP1twSjBx_VBpyfMsOY9VerPFVfnA%40mail.gmail.com), [#31 (JoongHyuk Shin, 2026-08-27)](https://www.postgresql.org/message-id/CACSdjfPCKNx9txxs5MGEhVUNC4H8nXkANgYAVR0%2Bb8kef_1P%2Bg%40mail.gmail.com), [#39 (Xuneng Zhou, 2026-10-10)](https://www.postgresql.org/message-id/CABPTF7XOitC_y0FNnm1hPKC%3DhZ6zL0kGU1W8W0MtS6iGOQX9DA%40mail.gmail.com), [#42 (shihao zhong, 2026-10-28)](https://www.postgresql.org/message-id/CAGRkXqRswzdS%2BWAQO_VoKi4zk5qso1Cop7x_2t53aq1QUuFEyg%40mail.gmail.com)

**Question:** Will v10, including the lock-conflict fix, be backported to every stable branch, and in what order will it land relative to CF 6445 and the related ResolveRecoveryConflictWithBufferPin patches?

Counter-check: held. The CommitFest entry targets stable, but the stable-branch patches stop at v6.1 (message #19) and lack the lock-conflict fix and the timeouts_armed flag from message #39. New stable-branch versions would close the gap, so it stays fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes how the startup process waits for buffer pins and locks during hot standby replay. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds exported functions to bufmgr.h, and the entry targets stable branches. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix targets REL_14_STABLE through REL_18_STABLE, but stable-branch patches exist only up to v6.1. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Five stable branches carry the new wait loop, and other patches touch ResolveRecoveryConflictWithBufferPin. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will v10, including the ResolveRecoveryConflictWithLock fix and the timeouts_armed flag, be posted for each stable branch from REL_14_STABLE to REL_18_STABLE?
2. In what order should this patch, CF 6445 and the other patches that touch ResolveRecoveryConflictWithBufferPin be committed?
3. Do the v10 TAP tests fail reliably on an unpatched server, given that they check the mechanism and not the hang?
4. Do the stable-branch versions only add functions to bufmgr.h, with no change to an existing exported struct or signature?
5. Should the patch add a comment in timeout.c saying that handle_sig_alarm can set the latch when no active timeout fired?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao, who committed the P1 revert in this code, reviewed in message #3. The brief says no committer has replied since June, so someone who knows the recovery conflict and buffer pin wait code must still agree to take it.
- **Liability:** The committer signs for the startup process wait loops for buffer pins and locks on master and five stable branches, with no time limit. A mistake there can stall replay or cancel queries wrongly on production standbys.
- **Author follow-up:** Good sign: the patch reached v10 over about nine months, with tests added and a co-author who also registered the CommitFest entry and reviews.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug stalls replay on hot standbys, most severely with max_standby_streaming_delay = -1, and several people reproduced it. The patch touches three files but must be carried on five stable branches, and the stable-branch versions need a refresh to v10 first.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Spurious-wakeup handling is fixed per call site, not at the source. Refuted (already handled): Messages #23 and #39 show the wakeups are not only from timeout.c: v10's tests trigger them with pg_log_backend_memory_contexts, so a timeout.c fix cannot do the job. Rechecking the condition after each latch wakeup is the existing Postgres convention, and the scenario concerns a hypothetical later patch.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected for a reason that applies here. P1 rested on the assumption that only an unpin or its own timeouts wake the startup process, and this patch removes that dependency instead of breaking it. P2 and P4 cover other conflict paths with different failure causes. The thread points to related work (message #18, #24, #26, CF 6445 in message #42) as pending patches that overlap with this one, but none of them is a rejection.
- standards-compat: 0 objection(s). The patch adds no SQL syntax, GUC, catalog object, dumped state or psql behaviour. The only user-visible change is that the documented deadlock_timeout cancellation on a standby now fires as the docs describe, which counts as a correction of existing behaviour rather than a new contract. New bufmgr.h functions in stable branches are additions and do not change existing signatures, so no standards or compatibility objection applies.
- replication-ondisk: 0 objection(s). The proposal changes only how the startup process waits for a buffer pin or lock during hot standby replay. It adds no WAL record, on-disk format, catalog, protocol or replication slot state, and the new bufmgr.h functions are additive, so back branches keep their ABI. Two points cannot be checked without the code: whether v10's timeouts_armed flag and lock-conflict change touch any exported struct, and whether they still apply to the stable branches, where backpatches exist only up to v6.1.
- security: 0 objection(s). The patch adds no new privilege, role, SQL function, GUC or monitoring output, and it does not let any user see more than before. It narrows an existing problem: a standby session that holds a buffer pin or lock can no longer stall replay forever when max_standby_streaming_delay = -1. pg_log_backend_memory_contexts appears only in the TAP tests, so the patch does not change who can wake the startup process.
- maintenance-fit: 2 objection(s). The bug is real. A committer who knows the area reviewed the patch in depth, and the core design was settled in the thread. The objections above are about long-run cost and the backpatch shape, not about whether to fix the bug.
- already-exists: 0 objection(s). This is a bug fix for a recovery stall. Postgres has no existing feature or extension that does the same job. Workarounds exist, such as a finite max_standby_streaming_delay, log_startup_progress_interval = 0, or deadlock_timeout below the progress interval. They only hide the defect, and a committer would not reject a bug fix because of them. v10-0003 removes the duplicate waiter registration in LockBufferForCleanup.
- demand: 0 objection(s). This is a bug fix for a confirmed hang. Recovery can stall indefinitely on a hot standby, and the fix targets back branches. Several people reproduced the bug independently (messages #1, #10, #39, #42), and the lock-conflict variant was found separately. The fix adds no user-facing surface, so the demand test does not apply and I see no objection from this angle.

### Precedents retrieved

- P1: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - used in the review
- P2: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - used in the review; reason summarised from its thread
- P3: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P4: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - used in the review; reason summarised from its thread
- P5: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P6: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P7: [Generalize ereport_startup_progress infrastructure](https://commitfest.postgresql.org/patch/3807/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P8: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - judged not relevant; reason summarised from its thread
- P9: [Add recovery_min_apply_delay_reconnect recovery option](https://commitfest.postgresql.org/patch/1325/) (cf, Returned with feedback, 2017-11-15) - judged not relevant; reason summarised from its thread
- P10: [WIP: Pgbench Serialization and deadlock errors](https://commitfest.postgresql.org/patch/1170/) (cf, Returned with feedback, 2017-09-28) - judged not relevant; reason summarised from its thread
- P11: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (revert, Reverted, 2017-04-24) - judged not relevant
- P12: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P13: [Avoiding deadlock errors in CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/2141/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P14: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P15: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P16: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant
- P17: [Pinning a buffer in TupleTableSlot is unnecessary](https://commitfest.postgresql.org/patch/755/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P18: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P19: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P20: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Use function smgrclose() to replace the loop](https://commitfest.postgresql.org/patch/5196/) (cf, Withdrawn, 2025-12-02) - judged not relevant
- P22: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P23: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P24: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 129 s; cost reported by the CLI: $2.97
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 55 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:46:50.012Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
