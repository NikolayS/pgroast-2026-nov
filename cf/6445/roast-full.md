# Design pre-review: Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6445](https://commitfest.postgresql.org/patch/6445/) (Needs review); authors: Anthony Hsu (erwaman)
- Thread: [Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin](https://www.postgresql.org/message-id/flat/CALQc50gi-Kw9m1r6hytf12473-fCECy%3Dq9JtKS4ANeJFEyCBTw%40mail.gmail.com), started 2025-07-06
- Messages read: 12 of 12
- Patch prose read: v2-0001-Set-1s-WaitLatch-timeout-if-standby-limit-has-exp.patch, v1-0001-Set-1s-WaitLatch-timeout-if-standby-limit-has-exp.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the race is stated step by step, and two production reports (messages #4 and #6) show demand. The one surviving objection, maintenance-fit-1, is a fixable level 2 point about commit order with the loop rewrite and the missing test. The fix changes how the startup process waits and targets stable branches, so a committer who knows recovery conflict handling has to sign for it.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, we could not check that the timed wait rechecks the pin count correctly, keeps the same wait event in pg_stat_activity, or wakes only on unpin and its own timeout as P1 requires.

## The proposal, restated

**Problem.** After the standby limit has expired, the startup process broadcasts one buffer pin cancel signal and then waits with no timeout. A backend that takes a conflicting pin after that broadcast never gets a signal, so replay can stall without limit. Hot standby users hit this; reporters saw it on 17 and 18, one with 5h41m of replay lag.

**Proposed design.** When the standby limit has already expired, the startup process waits on its latch with a 1s timeout instead of an unbounded wait. On timeout, LockBufferForCleanup() rechecks the pin count and calls ResolveRecoveryConflictWithBufferPin() again, which resends the conflict signal. v3 calls WaitLatch directly in standby.c and leaves proc.c and proc.h untouched, so backpatching is easier.

**What the author claims.**
- The race lets a late pinner block replay for an arbitrarily long time, because no second cancel signal is ever sent.
- With the fix, a late conflicting backend is cancelled about one second after it takes the pin.
- v3 does not touch proc.c or proc.h, so it should be easy to backpatch.
- The stall occurs in production: Dmytro on 18.4, Andrey on 17.11 with 5h41m lag.
- The fix is still needed after the patch that rewrites the same wait as a loop.

**Thread state.** Shihao, Andrey and João tested the fix and report it works on master and REL_17_STABLE. Xuneng accepts the diagnosis but would prefer an event-driven approach to repeated broadcasts, with no concrete design; tests, comment placement and the duplicate CF entry 6444 are still open.

## 2. Prior attempts

### Revert "Get rid of the dedicated latch for signaling the startup process". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7)
- **Thread:** [https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com), [https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com), [https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi](https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi)
- **Date:** decided 2020-12-17; started 2020-11-04
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert ac22929a26, as well as the followup fix 113d3591b8. Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin(). It caused, for example, SIGHUP signal handler or walreceiver process to wake that startup process up unnecessarily frequently. Additionally, add the comments about why that dedicated latch that the reverted patch tried to get rid of should not be removed. Thanks to Kyotaro Horiguchi for the discussion.
- **Reason (quote):** "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()."
- **What is different this time:** P1 removed the dedicated startup latch, so unrelated events woke the startup process during a buffer pin wait. v3 adds a timeout set inside ResolveRecoveryConflictWithBufferPin() itself, which is one of the two wakeup sources the revert allows.


## 3. Objections a reviewer is likely to raise

### 1. Fix and the pending loop rewrite of the same wait can drift apart

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), backpatchability, WAL / replication / standby · `maintenance-fit-1`

v3 is committed and backpatched to 17 and 18. Later, the other patch that rewrites this wait as a loop lands on master, and its expired path still sleeps with no timeout. The late-pinner stall comes back on master only, with no test to catch it, and users find it again through replay lag in production.

Message #5 says the two patches do not apply together and that the loop rewrite still needs this fix. The material does not say which patch goes first or who carries the timeout into the loop. The patch adds no test, and message #8 calls a TAP test possible but timing-sensitive, so a regression here would show up only as replay lag. Release branches already differ in signal and wait-event names (message #6), so master and the stable branches start out with different code.

Thread: [#5 (shihao zhong, 2026-10-02)](https://www.postgresql.org/message-id/CAGRkXqSn-PU-KwU6GwwhhJxZ8yyjioRf-kxzR0UrS5y_Hbk3yA%40mail.gmail.com), [#6 (Andrey Borodin, 2026-10-08)](https://www.postgresql.org/message-id/A65E3EDC-92B5-4C69-818F-0089F7AEB428%40yandex-team.ru), [#8 (João Rito, 2026-11-10)](https://www.postgresql.org/message-id/CADMs5OMJK%3DjaQJyePDfzDarvXo5i0gMLO8trZPgkbJJ6iJzGog%40mail.gmail.com)

**Question:** In which order will this fix and the loop rewrite be committed, and what test or check stops the expired path from losing its timeout again on master?

Counter-check: held. Message #5 confirms the two patches conflict and that the loop rewrite's expired path has no timeout; the thread sets no order and the patch has no test. The differing names on release branches are ordinary backpatch work and should not count as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes how the startup process waits for a cleanup lock during WAL replay on a hot standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CF entry targets stable branches, and v3 avoids proc.c and proc.h to ease backpatching; no range is agreed. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fix conflicts with a pending loop rewrite of the same wait and has no regression test. |
| performance on the hot path | **yes** | The startup process resends the conflict signal to all backends once per second during an expired conflict; nobody measured the cost. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. In which order will this fix and the loop rewrite of the same wait be committed, and who carries the 1s timeout into the loop?
2. What test or check would catch the expired path losing its timeout again, given that message #8 calls a TAP test timing-sensitive?
3. Which stable branches should get the fix, and does v3 apply to each of them given the differing signal and wait-event names in message #6?
4. What does the once-per-second broadcast cost on a standby with many backends during a long stall?
5. Which of CF entries 6444 and 6445 stays open for this thread?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows recovery conflict handling and the startup process latch. Álvaro Herrera created the CF entry in message #2 but did not review the design in the visible thread.
- **Liability:** The owner signs for a changed startup process wait on all backpatched branches, with no regression test, and for keeping the timeout when the loop rewrite lands.
- **Author follow-up:** The author rebased and posted v2 after the CF entry was created (message #3) and later posted v3 aimed at easier backpatching.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is small, lives in standby.c, and has production reports on 17 and 18 plus three testers. It has waited since 2025-07 through two CommitFests without a committer review of the design.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a reason that applies here. P1's rule allows a timeout set in ResolveRecoveryConflictWithBufferPin(), and v3 adds exactly that. P5 and P9 concern a different wait and a backend blocked in ClientWrite, and their reasons do not carry over to this fix.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, psql feature or dump state, and the 1s interval is a hardcoded define. The one visible behaviour change is in a minor release: a late pinner on a standby is now cancelled about one second after it takes the pin, where before it could hold replay without limit. That matches the documented contract of max_standby_streaming_delay, and Xuneng withdrew the concern that it breaks that contract in message #11. Not checked without the code: whether the wait event shown in pg_stat_activity stays the same during the timed wait.
- replication-ondisk: 0 objection(s). The proposal adds no persistent or cross-node state. It changes no WAL record, catalog, on-disk format or protocol message, and v3 leaves proc.h untouched, so it adds no ABI concern for backpatching. [P1] needed the startup process to be woken only by unpin or by its own timeout in this wait; a bounded latch wait in standby.c keeps that rule. The cost of the once-per-second broadcast belongs to the performance angle.
- security: 0 objection(s). The patch adds no new privilege, function, view or setting; it only bounds an existing startup-process wait, and it narrows a denial of service in which any standby user who takes a late buffer pin can stall replay. The once-per-second broadcast while the limit is expired affects only performance, and the material does not show any privilege or information-exposure aspect.
- maintenance-fit: 1 objection(s). From a maintenance and fit view this is a small, contained bugfix with production reports on 17 and 18 and several testers. The once-per-second broadcast cost was raised and weighed in message #8. [P1] supports a timeout-based wakeup, which this proposal uses, so it adds no objection.
- already-exists: 0 objection(s). This is a bug fix for a replay stall, and no existing feature fixes it. The thread names two ways out of a stall that has already happened: cancel the late pinner by hand, or run pg_log_backend_memory_contexts() on the startup process. Both need someone to notice the stall first, and Andrey saw 5h41m of lag in message #6. idle_in_transaction_session_timeout ends the idle cursor in the reproducer, but it does not help when the late pinner is an active query. The rewrite-as-loop patch in message #5 does not cover this case either, because its expired path still waits with no timeout. No extension or hook can change how the startup process waits. Duplicate CF entries 6444 and 6445 are a process question, not an existing mechanism.
- demand: 0 objection(s). Demand is shown. Two independent production reports exist: Dmytro saw it several times on 18.4 (message #4) and Andrey saw 5h41m replay lag on 17.11 (message #6). Several reviewers reproduced it, the cost falls only on standbys already in an expired conflict, and the only workaround is a manual superuser wakeup.

### Precedents retrieved

- P1: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - used in the review
- P2: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P3: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P4: [Pinning a buffer in TupleTableSlot is unnecessary](https://commitfest.postgresql.org/patch/755/) (cf, Returned with feedback, 2017-02-01) - judged not relevant; reason summarised from its thread
- P5: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - used in the review; reason summarised from its thread
- P6: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P7: [Inconsistency in startup process's MyBackendId and procsignal array registration with ProcSignalInit()](https://commitfest.postgresql.org/patch/3357/) (cf, Withdrawn, 2022-02-27) - judged not relevant; reason summarised from its thread
- P8: [Move PinBuffer and UnpinBuffer to atomics](https://commitfest.postgresql.org/patch/370/) (cf, Returned with feedback, 2015-12-16) - judged not relevant; reason summarised from its thread
- P9: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - used in the review; reason summarised from its thread
- P10: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P11: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P12: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P13: [Startup Centralization](https://commitfest.postgresql.org/patch/2498/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P14: [Revert "Move portal pinning from PL/pgSQL to SPI"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b48b2f8793ef256d19274b4ef6ff587fd47ab553) (revert, Reverted, 2018-01-10) - judged not relevant
- P15: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P16: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P17: [recovery modules](https://commitfest.postgresql.org/patch/4082/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P18: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P19: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P20: [New recovery.conf API](https://commitfest.postgresql.org/patch/774/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P21: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant
- P22: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P23: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P24: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 119 s; cost reported by the CLI: $1.72
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 49 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:52:13.313Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
