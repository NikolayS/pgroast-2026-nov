# Design pre-review: Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6444](https://commitfest.postgresql.org/patch/6444/) (Needs review); authors: not given
- Thread: [Set 1s WaitLatch timeout if standby limit has expired in ResolveRecoveryConflictWithBufferPin](https://www.postgresql.org/message-id/flat/CALQc50gi-Kw9m1r6hytf12473-fCECy%3Dq9JtKS4ANeJFEyCBTw%40mail.gmail.com), started 2025-07-06
- Messages read: 12 of 12
- Patch prose read: v2-0001-Set-1s-WaitLatch-timeout-if-standby-limit-has-exp.patch, v1-0001-Set-1s-WaitLatch-timeout-if-standby-limit-has-exp.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the panel at levels 1a, 1b or 2. The problem is stated with a concrete race, two independent field reports support it (messages #4 and #6), and the fix is a bounded timeout confined to standby.c. The open items are a test, the backpatch branch list, the unmeasured broadcast cost and the order relative to the parallel loop rewrite; they belong to committability and later review levels.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and v3 is known only from the brief. The recheck of the pin count after a timeout, the latch reset order and the interaction with the deadlock-check timeout could not be judged.

## The proposal, restated

**Problem.** After max_standby_streaming_delay expires, the startup process sends the buffer pin conflict signal once and then waits with no timeout. A backend that pins the buffer after that signal is never told to cancel, so replay can stall indefinitely. Dmytro saw it on 18.4 and Andrey on a 17.11 production standby, with lag reaching 5h41m.

**Proposed design.** When the standby limit has already expired, the startup process waits on its latch with a 1s timeout instead of waiting until UnpinBuffer() wakes it. On timeout it rechecks the pin count and sends the conflict signal again, so late pinners get canceled. v3 calls WaitLatch directly in standby.c and leaves proc.c and proc.h unchanged.

**User-visible surface.**
- Behavior change: after the standby limit expires, backends holding a conflicting pin are canceled about one second after they take it, instead of possibly never.

**What the author claims.**
- A backend can pin the buffer after the startup process sent the conflict signal and never receive another one, so replay stalls.
- A 1s timeout on the expired path lets the startup process resend the signal and cancel new conflicting backends.
- v3 does not touch proc.c or proc.h, so backpatching should be easy (Shihao).
- Reproduced on 18.4 and 17.11; v3 cancels the late pinner in about one second (Dmytro, Andrey, João).
- With max_standby_streaming_delay = -1 the wait stays unbounded, and the non-expired path is unchanged (João).

**Thread state.** Shihao, Andrey and João tested v3 and support it, and João asked for a test, commit trailers and a backpatch line. Xuneng accepts the diagnosis but worries about repeated broadcasts and would prefer an event-driven fix, with no concrete design yet.

## 2. Prior attempts

### Revert "Get rid of the dedicated latch for signaling the startup process". (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7)
- **Thread:** [https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com), [https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACXPorUqePswDtOeM_s82v9RW32E1fYmOPZ5NuE+TWKj_A@mail.gmail.com), [https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi](https://www.postgresql.org/message-id/flat/5c1f8a85-747c-7bf9-241e-dd467d8a3586@iki.fi)
- **Date:** decided 2020-12-17; started 2020-11-04
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert ac22929a26, as well as the followup fix 113d3591b8. Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin(). It caused, for example, SIGHUP signal handler or walreceiver process to wake that startup process up unnecessarily frequently. Additionally, add the comments about why that dedicated latch that the reverted patch tried to get rid of should not be removed. Thanks to Kyotaro Horiguchi for the discussion.
- **Reason (quote):** "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()."
- **What is different this time:** P1 removed the dedicated wakeup latch, so unrelated events woke the startup process. v3 keeps that latch and adds only a timeout inside ResolveRecoveryConflictWithBufferPin(), which is one of the two wakeup sources the revert message allows.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes how the startup process waits during a buffer pin recovery conflict on a hot standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v1 and v2 touch proc.c and proc.h; v3 is reported to leave both unchanged. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch narrows an existing way for any standby session to stall replay with a late buffer pin. |
| backpatchability | **yes** | The CF entry targets stable branches, and v3 needs signal and wait-event names adapted on 17. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a timed wait in recovery conflict code that few people know, and a parallel patch rewrites the same wait. |
| performance on the hot path | **yes** | The startup process resends the conflict signal to backends every second while replay is blocked after the limit expires. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which stable branches should receive the fix, and what does the adapted patch look like on 17, where the signal and wait-event names differ?
2. What does one conflict broadcast per second cost on a standby with many backends while replay stays blocked?
3. Should this fix go in before or after the parallel patch that rewrites the same wait as a loop, and which one gets rebased?
4. Can a test reproduce the late pin deterministically, for example with an injection point, without depending on timing?
5. Which of the CommitFest entries 6444 and 6445 stays open for this thread?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the startup process, recovery conflict handling and the buffer pin wakeup path. Álvaro Herrera created the CF entry in message #2 but has not reviewed the patch in the visible thread.
- **Liability:** A behavior change in recovery conflict handling on every supported branch that gets the backpatch, with no regression test so far. The owner also signs for the resend cost on busy standbys and for the merge with the parallel loop rewrite.
- **Author follow-up:** The author rebased and posted v2 in message #3 after seven months of silence on the thread, and a v3 followed. Shihao Zhong moved the entry to the next CommitFest and stays as reviewer.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is small and confined to standby.c, and two field reports show replay stalls of up to 5h41m on hot standbys. Three reviewers tested v3, but no committer who knows the area has reviewed it, and the entry has been open across two CommitFests.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed this fix, and no recorded reason applies against it. P1 is the closest: its rule allows a timeout set in ResolveRecoveryConflictWithBufferPin(), which v3 adds. P2 and P4 cover different conflict paths, and P11 bears only on how stable a test would be. Message #5 cites a parallel patch that rewrites the same wait as a loop and keeps the expired path untimed, so the two patches do not apply together.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog object, dump content or psql change. After expiry, late conflicting queries get canceled, as max_standby_streaming_delay already documents, and Xuneng withdrew the 'promise' concern in message #11, so the backpatched behavior change matches the stated contract.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no WAL record, on-disk format, catalog, protocol message or proc.h struct, so no pg_upgrade, mixed-version or extension ABI issue arises. [P1] was reverted for waking the startup process from unrelated sources, but this timeout is bounded and only applies after the standby limit has expired.
- security: 0 objection(s). The proposal adds no new privilege, role, setting, view or message, so it creates no new security exposure. It shrinks an existing denial-of-service path: today any user who can connect to a hot standby can stall replay with a late buffer pin (message #4), and v3 cancels that session in about a second. The extra cost is one conflict broadcast per second, and it only runs while replay is already blocked. A user who keeps pinning again could already hold up replay before this patch, so this is not a new attack.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle, this is a small bugfix confined to standby.c, and v3 leaves proc.c and proc.h alone (message #5). It bounds a wait that already has a 1s deadlock-check wakeup before expiry, and that wait only happens on a standby already stalled by a conflict. The extra cost is one conflict signal per second per stalled buffer. Backends that hold no conflicting pin pay little for it. There is no new GUC, no API, no on-disk or WAL change, and core is the only place this can be fixed. The extra wakeup comes from a timeout, which keeps the startup process woken only by unpin or a timeout. That fits the assumption [P1] protects. Xuneng's event-driven alternative has no design yet (message #12), and changing PinBuffer would add hot-path cost (message #10). Open items do not support a rejection. No committer has engaged beyond Álvaro creating the CF entry. The order relative to the loop rewrite in [1] is not settled. There is no test, the backpatch branch list is not stated, and the broadcast cost on standbys with many backends is not measured. These belong under committability and level 4.
- already-exists: 0 objection(s). Postgres has no existing feature, setting or extension that resends the buffer pin conflict signal once the standby limit has expired. The workarounds in the thread, a manual pg_terminate_backend or a diagnostic signal to the startup process (messages #4, #6), need an operator to notice a stall that can last hours, so they do not meet the need. The fix sits inside the startup process wait in standby.c, where a hook or an extension cannot reach. The bounded wait reuses the existing WaitLatch and conflict-signal path and adds no second mechanism.
- demand: 0 objection(s). The demand is shown in the thread. It is a bugfix for a replay stall with independent field reports: Dmytro saw it several times on 18.4 (message #4), and Andrey saw it on a 17.11 production standby with 5h41m lag (message #6). The only workaround is a manual superuser wakeup of the startup process, and the cost of the fix falls only on standbys whose limit has already expired.

### Precedents retrieved

- P1: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - used in the review
- P2: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - used in the review; reason summarised from its thread
- P3: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P4: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - used in the review; reason summarised from its thread
- P5: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P6: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P7: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P8: [Simplify standby state machine a bit in WaitForWALToBecomeAvailable()](https://commitfest.postgresql.org/patch/3965/) (cf, Withdrawn, 2023-03-03) - judged not relevant; reason summarised from its thread
- P9: [Inconsistency in startup process's MyBackendId and procsignal array registration with ProcSignalInit()](https://commitfest.postgresql.org/patch/3357/) (cf, Withdrawn, 2022-02-27) - judged not relevant; reason summarised from its thread
- P10: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P11: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - used in the review
- P12: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P13: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P14: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P15: [Pinning a buffer in TupleTableSlot is unnecessary](https://commitfest.postgresql.org/patch/755/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P16: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P17: [Microsecond-based timeouts](https://commitfest.postgresql.org/patch/4245/) (cf, Returned with feedback, 2023-07-26) - judged not relevant
- P18: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P19: [Start WAL receiver eagerly](https://commitfest.postgresql.org/patch/3376/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P20: [Revert unintentional change in increasing usage count during pin of buffers, this makes buffer access strategy have no effect.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cf6033281fdaf938b826545a643f951086d8671) (revert, Reverted, 2017-03-20) - judged not relevant
- P21: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P22: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant
- P23: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P24: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 97 s; cost reported by the CLI: $1.56
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:52:10.208Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
