# Design pre-review: ReplicationSlotRelease() clobbers another backend's statusFlags entry

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7366](https://commitfest.postgresql.org/patch/7366/) (Needs review); authors: Vlad Lesin (vlesin)
- Thread: [ReplicationSlotRelease() clobbers another backend's statusFlags entry](https://www.postgresql.org/message-id/flat/889a06dd-45fc-423b-9dd2-87d5b5dc60c6%40gmail.com), started 2026-09-28
- Messages read: 5 of 5
- Patch prose read: v2-0001-Fix-clobbering-of-shared-statusFlags-when-a-slot-.patch, v2-0002-Assert-that-a-PGPROC-is-in-the-proc-array-before-.patch, v1-0001-Fix-clobbering-of-shared-statusFlags-when-a-slot-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. Message #1 states the bug, the affected processes and the branch range, and Kuroda confirmed the analysis in message #2. v2-0001 changes one function and adds no user-visible surface. The open points are whether v2-0002 and the TAP test are committed, which the thread is already settling.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that only StartupDecodingContext() sets PROC_IN_LOGICAL_DECODING is taken from the commit message. Messages #4 and #5 were not shown; the committer's plan is taken from the brief.

## The proposal, restated

**Problem.** ReplicationSlotRelease() writes MyProc->statusFlags to ProcGlobal->statusFlags[MyProc->pgxactoff], but auxiliary processes are not in the proc array and have pgxactoff 0. When the checkpointer or startup process invalidates a slot, it overwrites the flags of the backend at offset 0. Assert builds crash, and losing PROC_AFFECTS_ALL_HORIZONS on a walsender can let vacuum remove rows a standby still needs.

**Proposed design.** v2-0001 skips the shared store in ReplicationSlotRelease() unless PROC_IN_LOGICAL_DECODING is set, and adds a TAP test; it is back-patched to v14. v2-0002 adds ProcArrayHasProc() and asserts it wherever a process changes its own statusFlags entry. v1 asserted that AmAuxiliaryProcess() is false instead.

**User-visible surface.**
- C API: new function ProcArrayHasProc() in procarray.h (v2-0002 only).
- C API: new macro AmAuxiliaryProcess() in miscadmin.h (v1 only, dropped in v2).
- New TAP test src/test/recovery/t/058_slot_invalidation_statusflags.pl.

**What the author claims.**
- Affects every branch from v14 through master; v13 passes the reproducer.
- A standby hits the bug during restartpoints, and from v16 during ordinary replay via 26669757b6a.
- From v18, idle_replication_slot_timeout can trigger it without max_slot_wal_keep_size.
- Nothing changes for processes in the proc array, since only StartupDecodingContext() sets PROC_IN_LOGICAL_DECODING.
- Core regression tests and the recovery and subscription TAP suites pass on master with the patch.

**Thread state.** Kuroda confirmed the analysis; Paquier plans to commit only v2-0001 down to v14 and doubts both the extra assertions and the test. The author replied that the test fails without the fix even without the v2-0002 assertions; CFbot reports the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The fix changes ReplicationSlotRelease() in slot.c, reached during slot invalidation by the checkpointer and the startup process on standbys. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v2-0002 adds ProcArrayHasProc() to procarray.h; v2-0001 adds no API. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | v2-0001 is marked Backpatch-through: 14, and the test needs a local advance_wal() helper before v17. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2-0002 adds assertions in six files, and the new TAP test must be kept working on six branches. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is there any production report of a standby losing rows through this path, or does the PROC_AFFECTS_ALL_HORIZONS case rest on analysis only?
2. Will the TAP test be committed with the fix, and what does the back-branch version use in place of $node->advance_wal() before v17?
3. Does test number 058 conflict with an existing test in src/test/recovery on any branch from v14 to v18?
4. Is v2-0002 meant for master only, and does any path besides ReplicationSlotRelease() reach its new assertions from a process outside the proc array?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier takes part in the thread and, per the brief, plans to commit v2-0001 down to v14. The owner needs to know replication slot invalidation and the proc array.
- **Liability:** A one-function change in ReplicationSlotRelease() on six branches, v14 through master. If the test is committed, the owner also signs for its stability on the buildfarm on each branch.
- **Author follow-up:** The author posted v2 one day after the first review, answered each review point in message #3, and replied again after the committer's comments.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug affects installations where the checkpointer or startup process invalidates a slot, on v14 and later, and can weaken hot standby feedback. v2-0001 is a small change in slot.c with a committer engaged; CFbot reports that the patch needs a rebase.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found; the corpus returned no row for the same or a similar idea. Message #1 points to a related v14-only report of the same assertion through a different route (a stale walsender pgxactoff), where nothing was committed, and to a single-user-mode fix of ReplicationSlotRelease(). Neither is a rejection of this idea, and the committer in message #4 plans to commit v2-0001 down to v14.
- standards-compat: 0 objection(s). Nothing in my angle applies. The fix changes no SQL syntax, GUC, catalog, pg_dump output, psql behaviour or client-visible protocol. The only behaviour change stops a write to another backend's statusFlags entry. The new ProcArrayHasProc() is an internal assertion helper in v2-0002, and the committer plans to skip that patch. Test numbering on back branches is a commit-time detail.
- replication-ondisk: 0 objection(s). The fix touches only shared-memory state in ProcGlobal->statusFlags. It changes no on-disk format, WAL record, catalog, wire protocol or back-branch ABI, and it removes a path that could weaken hot standby feedback. v2-0002 adds ProcArrayHasProc(), but it is master-only and Paquier plans to commit just v2-0001.
- security: 0 objection(s). The fix adds no new privilege, role, setting or user-visible surface; it removes a wrong shared-memory store made by the checkpointer or startup process. Reaching the bug path takes slot setup and server configuration, and the fix only narrows exposure, so the security angle has no objection.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a narrow bug fix in one function: v2-0001 adds no API, GUC, catalog change or hot-path cost. Michael Paquier, a committer in this area, plans to commit only v2-0001 down to v14 (message #4), which settles the v2-0002 assertions and the new ProcArrayHasProc() API. The back-branch test, which needs a local advance_wal() helper before v17, is a level-4 matter and outside scope.
- already-exists: 0 objection(s). This is a bug fix for a wrong-index store in ReplicationSlotRelease(), and no existing feature, setting or extension can stand in for a correct core fix. The added ProcArrayHasProc() assertion helper is closest to duplicating existing checks, but it is not a second mechanism for a user-facing job, and Paquier already plans to drop it (message #4).
- demand: 0 objection(s). This is a bug fix, so demand is not in doubt. The author found it from a real CI crash on a primary while autovacuum ran (message #1), Kuroda reproduced and confirmed it (message #2), and a committer plans to back-patch v2-0001 to v14 (message #4). The fix adds no user-visible surface. Whether v2-0002 and the test belong in the commit is a design and test question, outside the demand angle.

### Precedents retrieved

- P1: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P2: [Restore lock level to update statusFlags](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dcfff74fb16622898a9d1df33e530f477caab095) (revert, Partially reverted, 2020-11-26) - judged not relevant
- P3: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P4: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P5: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P6: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P7: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P8: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P11: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant; reason summarised from its thread
- P12: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant; reason summarised from its thread
- P13: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P14: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P16: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P17: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P18: [add assertion for palloc in signal handlers](https://commitfest.postgresql.org/patch/6498/) (cf, Withdrawn, 2026-08-24) - judged not relevant
- P19: ["Causal reads" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/487/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P20: [walsender.c fileheader comment](https://commitfest.postgresql.org/patch/5055/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P21: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P22: [Propagate sanity checks of ProcessUtility() to standard_ProcessUtility()?](https://commitfest.postgresql.org/patch/4877/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P23: [Startup Centralization](https://commitfest.postgresql.org/patch/2498/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P24: [Track replay lag in pg_stat_replication](https://commitfest.postgresql.org/patch/920/) (cf, Returned with feedback, 2017-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 76 s; cost reported by the CLI: $0.96
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 45 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:46:08.206Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
