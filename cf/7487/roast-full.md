# Design pre-review: Fix recovery connection detail in EXEC_BACKEND builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7487](https://commitfest.postgresql.org/patch/7487/) (Needs review); authors: Nitin Jadhav (nitinjadhav11)
- Thread: [\[PATCH\] Fix recovery connection detail in EXEC_BACKEND builds](https://www.postgresql.org/message-id/flat/CAMm1aWYtiKDv7uzPWJZ2Gre2973bRV%3DGTOhd6BU_-VxdeSkGVg%40mail.gmail.com), started 2026-10-12
- Messages read: 4 of 4
- Patch prose read: v2-0001-fix-recovery-connection-detail-in-exec-backend-builds.patch, v1-0001-fix-recovery-connection-detail-in-exec-backend-builds.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated in message #1, the fix makes EXEC_BACKEND builds report the same DETAIL and HINT as fork() builds, and v2 reuses the existing CAC_state mechanism as Heikki Linnakangas suggested in message #2. Confidence is medium because the code was not read and nobody has replied to v2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked whether all CAC_state consumers handle CAC_NOTCONSISTENT or where the postmaster reads the consistency state. The author did not test on Windows.

## The proposal, restated

**Problem.** In EXEC_BACKEND builds the backend does not inherit reachedConsistency from the postmaster. A connection rejected after consistency but before the hot standby snapshot is ready therefore gets the wrong DETAIL ("Consistent recovery state has not been yet reached") and no hint about transactions with more than 64 subtransactions. Users of EXEC_BACKEND builds, such as Windows, see the misleading message.

**Proposed design.** v2 restores the CAC_NOTCONSISTENT code and has the postmaster pick it or CAC_NOTHOTSTANDBY before it launches the backend. The backend then chooses DETAIL and HINT from CAC_state alone. v1 instead passed reachedConsistency through BackendStartupData, and Heikki suggested the change to a separate CAC_* code.

**User-visible surface.**
- Connection rejection DETAIL and HINT during hot standby startup change in EXEC_BACKEND builds to match fork() builds.
- Internal enum CAC_state gains CAC_NOTCONSISTENT again (src/include/tcop/backend_startup.h).

**What the author claims.**
- Only the reported DETAIL and HINT change; connection acceptance and recovery behavior do not.
- Core regression tests pass on Linux in normal and EXEC_BACKEND builds.
- Manual testing showed the expected DETAIL and HINT.
- The author did not test on Windows.

**Thread state.** Heikki Linnakangas reviewed v1 and suggested a new CAC_* code, and the author posted v2 that does this. Nobody has replied to v2 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-12; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes the DETAIL and HINT of connections rejected on a standby during hot standby startup; recovery behaviour is unchanged. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds CAC_NOTCONSISTENT to the CAC_state enum in src/include/tcop/backend_startup.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix but the thread does not say which branches get it; a back-patch changes an enum in a header. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches should get the fix, and if back-patched, is CAC_NOTCONSISTENT added at the end of CAC_state so existing values keep their numbers?
2. Can the postmaster's view of consistency be out of date when it picks CAC_NOTCONSISTENT or CAC_NOTHOTSTANDBY, and does that differ from fork() builds today?
3. Does every remaining consumer of CAC_state handle the restored CAC_NOTCONSISTENT value?
4. Can a TAP test cover the DETAIL and HINT for a connection rejected after consistency but before the hot standby snapshot is ready?
5. Has anyone run v2 on Windows, the main platform that uses EXEC_BACKEND?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas reviewed v1 in message #2 and suggested the design that v2 uses. Nobody has replied to v2 yet.
- **Liability:** Small. The owner signs for one restored CAC_state value and the postmaster's choice between two rejection codes, plus the back-patch decision.
- **Author follow-up:** The author answered the review in three days in message #3 and posted v2 as promised.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix affects only EXEC_BACKEND builds, such as Windows, during hot standby startup. The patch is about 4 KB across three files and removes the backend's reliance on an inherited postmaster variable.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at the same or a similar idea. Message #1 points to a recent reachedConsistency discussion that mentioned this bug, and message #3 says CAC_NOTCONSISTENT existed before, but neither reports a rejection or revert reason that would apply here.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, GUC, catalog object, dump content or psql change. The only user-visible change makes the DETAIL and HINT in EXEC_BACKEND builds match what fork() builds already report. CAC_state is an internal enum, so restoring CAC_NOTCONSISTENT has no visible effect on clients. The thread does not say which branches get the fix. Clients that match on DETAIL text may see a different message after a minor upgrade, but that message is already the documented behaviour on other platforms.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state: no WAL record, on-disk format, catalog, protocol message or replication behavior changes. CAC_state is postmaster-to-backend startup data in the same build. If backpatched, appending the restored enum value avoids renumbering, but that is a detail for the committer and not a design objection.
- security: 0 objection(s). The patch changes only which DETAIL and HINT a rejected connection gets during hot standby startup in EXEC_BACKEND builds. Fork() builds already show the same text to unauthenticated clients, so nothing new leaks, and the change touches no privilege, authentication or resource limit.
- maintenance-fit: 0 objection(s). No maintenance or fit objection stands. This is a small bugfix that removes the backend's reliance on an inherited postmaster global; this pattern is a known source of EXEC_BACKEND bugs, so the change should lower future maintenance. The postmaster already computes CAC_state per connection, so non-users pay no per-connection or hot-path cost. Heikki Linnakangas, a committer who knows this area, shaped v2 in message #2. Not checked without the code: whether every remaining CAC_state consumer handles the restored CAC_NOTCONSISTENT value.
- already-exists: 0 objection(s). This is a bug fix for a wrong DETAIL and HINT in EXEC_BACKEND builds. No existing feature, setting or extension gives the correct message, so the need is not already met. The v2 design reuses the existing CAC_state mechanism. It does not add a second mechanism, so nothing from this angle applies.
- demand: 0 objection(s). This is a small bug fix: Windows and other EXEC_BACKEND builds report the wrong DETAIL and drop a HINT, so the message differs from fork() builds. A correctness fix for a misleading error message on a supported platform does not need a separate demand argument, and its cost is limited to one restored CAC_state code.

### Precedents retrieved

- P1: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P2: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P3: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P4: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P5: [postgres_fdw - make cached connection functions tests meaningful with debug_invalidate_system_caches_always](https://commitfest.postgresql.org/patch/3112/) (cf, Rejected, 2021-07-21) - judged not relevant; reason summarised from its thread
- P6: [Close backend file handles on exec](https://commitfest.postgresql.org/patch/886/) (cf, Returned with feedback, 2017-01-31) - judged not relevant; reason summarised from its thread
- P7: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant; reason summarised from its thread
- P8: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P9: [Add function - pg_accept_connections_start_time()](https://commitfest.postgresql.org/patch/5576/) (cf, Returned with feedback, 2025-07-25) - judged not relevant; reason summarised from its thread
- P10: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P11: [Reorganize CacheMemoryContext](https://commitfest.postgresql.org/patch/5337/) (cf, Withdrawn, 2024-10-29) - judged not relevant
- P12: [Direct SSL Connections](https://commitfest.postgresql.org/patch/4135/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P13: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant
- P14: [REINDEX backend filtering](https://commitfest.postgresql.org/patch/2895/) (cf, Withdrawn, 2021-05-08) - judged not relevant
- P15: [Bug: wrong relname in RemoveSubscriptionRel() error detail](https://commitfest.postgresql.org/patch/6621/) (cf, Withdrawn, 2026-07-12) - judged not relevant
- P16: [Have BackendXidGetPid return pid_t](https://commitfest.postgresql.org/patch/6229/) (cf, Rejected, 2026-01-07) - judged not relevant
- P17: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P18: [REINDEX filtering in the backend](https://commitfest.postgresql.org/patch/2206/) (cf, Returned with feedback, 2019-09-06) - judged not relevant
- P19: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P22: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P23: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant
- P24: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 49 s; cost reported by the CLI: $0.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:00:21.118Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
