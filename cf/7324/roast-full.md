# Design pre-review: [Patch] Fix pg_get_multixact_stats() over-reporting members on a hot standby

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7324](https://commitfest.postgresql.org/patch/7324/) (Needs review); authors: Naga Appani (naga.appani)
- Thread: [\[Patch\] Fix pg_get_multixact_stats() over-reporting members on a hot standby](https://www.postgresql.org/message-id/flat/CA%2BQeY%2BDcnsZRL4GZD8JyJu5HD-d6p8_9M5kT4jgF3gq3Rh8n4w%40mail.gmail.com), started 2026-09-21
- Messages read: 8 of 8
- Patch prose read: v1-0001-Maintain-MultiXact-oldestOffset-during-recovery.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated and reproduced in message #1, and the thread agreed in messages #2 and #3 to raise an error during recovery, which copies the existing xlogfuncs.c practice. The function is new in 19, so no released behaviour changes. Confidence is medium because v2, which carries the agreed design, is not attached.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: v2 is not attached, so its error check, documentation and tests could not be judged. Whether v1's redo-path update of oldestOffset remains in the final patch, and whether it is safe during replay, also needs the code.

## The proposal, restated

**Problem.** On a hot standby, MultiXactState->oldestOffset is not maintained and stays at 0, so pg_get_multixact_stats() reports num_members as nextOffset. num_members and members_size are too large, while num_mxids and oldest_multixact are correct. This misleads anyone who monitors multixact member space on a standby.

**Proposed design.** v1 updates oldestOffset while replaying XLOG_MULTIXACT_TRUNCATE_ID, and TrimMultiXact() still recomputes it at end of recovery. The value still reads 0 until the first truncation record arrives and again after a standby restart. The thread then agreed to raise an error when the function runs during recovery, which the author says matches an unposted v2.

**User-visible surface.**
- pg_get_multixact_stats(): num_members and members_size become correct on a standby once a truncation record is replayed (v1).
- pg_get_multixact_stats(): raises an ERROR with ERRCODE_OBJECT_NOT_IN_PREREQUISITE_STATE when called during recovery (agreed direction, v2).
- Error message and hint reused from xlogfuncs.c, so no new translatable strings.

**What the author claims.**
- On REL_19_BETA3 the standby showed about 17.5M members and 84 MB while the primary showed zero for the same data.
- With v1 the standby and primary return matching results for the same workload.
- Promoting the standby fixes the values at once, because oldestOffset is computed at end of recovery.
- Multixact numbers are not actionable on a standby, since freezing and vacuum run on the primary, so an error is more honest than wrong numbers.

**Thread state.** Heikki and Michael replied; Heikki rejected the redo-only fix because the window can be long and returns after restart. Michael said he would make the function error during recovery, and the author says v2 already does this.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | v1 changes replay of XLOG_MULTIXACT_TRUNCATE_ID in multixact.c, and the agreed v2 changes the function's behaviour on a standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is a bug fix targeted at 19, where pg_get_multixact_stats() is new and still in beta. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can you post v2, so reviewers can see the exact error behaviour of pg_get_multixact_stats() during recovery?
2. Does v2 keep v1's update of oldestOffset while replaying XLOG_MULTIXACT_TRUNCATE_ID, or does it drop that change?
3. Will the documentation of pg_get_multixact_stats() state that the function raises an error during recovery?
4. Does v2 add a test that calls pg_get_multixact_stats() on a standby and checks the error?
5. Is the loss of the correct num_mxids and oldest_multixact on a standby acceptable, given that only num_members and members_size are wrong today?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas and Michael Paquier replied in the thread, and Michael said he would make the function raise an error during recovery.
- **Liability:** Small: one recovery check in a monitoring function that is new in 19, with an error message reused from xlogfuncs.c. The deferred control-file field for oldestOffset stays open for a later release.
- **Author follow-up:** The author says in message #1 that he wants to help but may be slow to respond, and invites others to take the patch over.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix corrects wrong output from a function that is new in 19, so it is best settled before release. The patch touches one file, the agreed design adds no new state or strings, and a committer has offered to make the change.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no row about this or a related idea. Message #1 points to an earlier discussion ([0]) where Heikki raised the bug; the thread reports only that he suggested the redo-path fix and a control-file start value, and nothing there reads as a rejection.
- standards-compat: 0 objection(s). The contract change touches only pg_get_multixact_stats(), which is new in 19 and not yet in a released branch, so no released query, dump or client changes behaviour. Heikki raised the loss of the correct num_mxids and oldest_multixact on a standby in message #4, and the thread chose an error in messages #3 and #6; the error reuses the xlogfuncs.c message and SQLSTATE that existing recovery-restricted functions already use. Not checkable without v2: whether the docs state that the function fails during recovery.
- replication-ondisk: 0 objection(s). The agreed direction (v2) makes pg_get_multixact_stats() raise an error during recovery. That changes no WAL record, control file field, catalog, on-disk format or wire message, so pg_upgrade, replay, mixed-version standbys and ABI are not affected. v1's redo-side update of oldestOffset in shared memory is superseded in the thread, and the control-file option that would need a catversion bump was deferred in message #4.
- security: 0 objection(s). The fix changes how an existing monitoring function computes or refuses to report multixact member counts on a standby. It adds no new privilege, data exposure, escalation path or unbounded resource use, and an error during recovery exposes less than before. Who may call pg_get_multixact_stats() could not be checked from the material, but the proposal does not change it.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The agreed direction, raising an error from pg_get_multixact_stats() during recovery, adds no hot-path work, no new state, no external dependency and no new translatable strings. It reuses the xlogfuncs.c error messages. The function is new in 19, so no released monitoring depends on it. Michael, a committer, said in message #7 that he will make the change, so ownership is clear. The open items are not maintenance or fit questions: whether v1's redo-path update is also kept, the docs and the tests, and the deferred control-file field.
- already-exists: 0 objection(s). No existing feature gives correct member counts on a standby. pg_control_checkpoint() exposes next_multi_offset but not the oldest offset, so this is a bug fix to a new core function rather than a duplicate. The agreed error during recovery copies the existing xlogfuncs.c practice and reuses its messages, so it adds no new mechanism.
- demand: 0 objection(s). This is a bug fix for wrong output from a new function, so it needs no demand evidence beyond the reproduced mismatch in message #1. The agreed direction, an error during recovery, adds no feature surface, and committers Heikki and Michael are engaged.

### Precedents retrieved

- P1: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P2: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P3: [Revert the addition of GetMaxBackends() and related stuff.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7fc0e7de9fb8306e84d1c15211aba4308f694455) (revert, Reverted, 2022-04-12) - judged not relevant
- P4: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant; reason summarised from its thread
- P5: [Simplify standby state machine a bit in WaitForWALToBecomeAvailable()](https://commitfest.postgresql.org/patch/3965/) (cf, Withdrawn, 2023-03-03) - judged not relevant; reason summarised from its thread
- P6: [SQL function which allows to distinguish a server being in point in time recovery mode and an ordinary replica](https://commitfest.postgresql.org/patch/5513/) (cf, Withdrawn, 2025-03-17) - judged not relevant; reason summarised from its thread
- P7: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant; reason summarised from its thread
- P8: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P9: [SQL function which allows to distinguish a server being in point in time recovery mode and an ordinary replica](https://commitfest.postgresql.org/patch/5611/) (cf, Returned with feedback, 2025-08-04) - judged not relevant; reason summarised from its thread
- P10: [Fix PGXS so built control files work with VPATH builds](https://commitfest.postgresql.org/patch/2461/) (cf, Returned with feedback, 2020-07-31) - judged not relevant; reason summarised from its thread
- P11: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P12: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P13: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P14: [Configurable location for extension .control files](https://commitfest.postgresql.org/patch/170/) (cf, Returned with feedback, 2015-07-30) - judged not relevant
- P15: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P16: [recovery modules](https://commitfest.postgresql.org/patch/4082/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P17: [Fixes for non-atomic read of read of control file on ext4 + ntfs](https://commitfest.postgresql.org/patch/4025/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P19: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2409/) (cf, Returned with feedback, 2020-01-28) - judged not relevant
- P20: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P21: [I thought it would be better to add a POSIx_fadvise description for the recovery_prefetch parameter here, to distinguish it from the database functions.](https://commitfest.postgresql.org/patch/3689/) (cf, Withdrawn, 2022-06-17) - judged not relevant
- P22: [Add recovery_min_apply_delay_reconnect recovery option](https://commitfest.postgresql.org/patch/1325/) (cf, Returned with feedback, 2017-11-15) - judged not relevant
- P23: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P24: [TRUNCATE on foreign tables](https://commitfest.postgresql.org/patch/2499/) (cf, Returned with feedback, 2020-08-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 62 s; cost reported by the CLI: $0.75
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:40:35.676Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
