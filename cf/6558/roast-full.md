# Design pre-review: Pgbench: remove synchronous prepare

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6558](https://commitfest.postgresql.org/patch/6558/) (Needs review); authors: Dmitrii Bondar (difesu)
- Thread: [Pgbench: remove synchronous prepare](https://www.postgresql.org/message-id/flat/2da56b59-acb6-4ee3-af6b-6e800115dc7f%40postgrespro.ru), started 2026-01-27
- Messages read: 15 of 15
- Patch prose read: v5-0001-Remove-synchronous-prepare-from-pgbench.patch, userlist.txt, 0001-Remove-synchronous-prepare-from-pgbench.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2, because the panel raised none. The problem is stated with a reproducer in message #1, and a second case without a pooler is reported in message #14. v5 stays inside pgbench.c and its TAP test and uses the existing PQsendPrepare(), which is what message #9 asked for. Confidence is medium because nobody has reviewed v5 in the visible thread and the code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the panel could not check that the new state has no blocking PQgetResult() left, handles connection loss, and keeps failure counting and --report-per-command timing unchanged.

## The proposal, restated

**Problem.** With -M prepared, pgbench prepares each command with the blocking PQprepare(), so one waiting prepare stalls every client on the same worker thread. The author hit this behind PgBouncer in session pooling mode with more clients than the pool size. A reviewer added that a prepare waiting on a table lock stalls unrelated clients the same way.

**Proposed design.** v5 replaces PQprepare() in sendCommand() with the asynchronous PQsendPrepare() and adds a new state, CSTATE_WAIT_PREPARE_RESULT, which waits for the result through the event loop. A new helper reads the sqlstate from an error result without reading from the connection. The pipeline path stays synchronous on purpose, and the earlier design with a new libpq function was dropped.

**User-visible surface.**
- pgbench -M prepared no longer blocks the worker thread while a Prepare waits.
- Error output on a failed prepare changes: the extra "prepared statement ... does not exist" error no longer follows a syntax error.
- No libpq API change in v5; v1 to v3 added PQsendPBES to libpq exports.

**What the author claims.**
- The fix makes -M prepared usable behind PgBouncer in session pooling mode when clients exceed the pool size.
- Prepare time stays in --report-per-command because v5 sends the query from the new state without resetting the command timer.
- A reviewer said the patch resolves the blocking with no regressions in their testing.
- A new TAP test shows one client progressing while another waits on a locked table during Prepare.

**Thread state.** Robert Haas rejected the new libpq entrypoint and asked for PQsendPrepare(); the author moved to a pgbench state machine change, which Andrey Borodin supports. v5 answers Borodin's review points, and no one has reviewed v5 in the visible thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-01-27; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | v5 adds the CSTATE_WAIT_PREPARE_RESULT state to the pgbench state machine and leaves the pipeline path synchronous. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does any blocking PQgetResult() call remain in the new prepare path, including when the connection fails between results?
2. Do retry and failure counts under --max-tries and --failures-detailed stay the same when a Prepare fails in the new state?
3. Is there a plan or a tracking item to give prepareCommandsInPipeline() the same asynchronous treatment?
4. Is the documentation change from message #2 still needed, given that v5 touches only pgbench.c and the test file?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Robert Haas took part in the thread and asked for PQsendPrepare(), but he has not commented on v5. The owner needs to know the pgbench client state machine and libpq asynchronous result handling.
- **Liability:** The owner signs for one more state in the pgbench state machine and for the changed error output on a failed prepare. The risk is limited to pgbench runs with -M prepared.
- **Author follow-up:** The author posted five versions over about eight months, dropped the libpq function after the objection in message #9, and answered the review points in v5.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix helps pgbench users who run -M prepared with fewer threads than clients, behind a pooler or under lock waits. v5 is a 14.7 KB patch to pgbench.c and one TAP test with no libpq change, and it waits for a review of v5.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier attempt at this idea. The thread mentions no earlier discussion of it. Within the thread, v4/v5 drops the libpq entrypoint that message #9 objected to and uses PQsendPrepare(), which is what messages #9 and #11 asked for, so no earlier rejection reason still applies.
- standards-compat: 0 objection(s). v5 changes only pgbench.c and its TAP test, with no new option, GUC, syntax or libpq export, so the user-facing contract stays as it is. The one visible change, dropping a follow-on "prepared statement does not exist" error after a syntax error, explains the real failure better; message #8 accounts for it and the test was updated. Without the code I could not check whether retry or failure counting under --max-tries and --failures-detailed changes for prepare errors.
- replication-ondisk: 0 objection(s). v5 touches only pgbench.c and its TAP test. It adds no on-disk, WAL, catalog, wire protocol or libpq ABI change, because the earlier PQsendPBES export was dropped. It uses existing protocol messages through PQsendPrepare(), so this angle finds nothing to object to.
- security: 0 objection(s). The proposal changes how pgbench, a client benchmark tool, waits for a Prepare result. It adds no privilege, role, GUC, catalog entry or server-side code, and v5 no longer changes the libpq API. The only privilege-related point is that pgbench runs with whatever rights its connection role has, and that does not change, so this angle has no objection.
- maintenance-fit: 0 objection(s). From the maintenance angle v5 is local to pgbench.c and its TAP test, adds no libpq export, and costs nothing for runs without -M prepared. Two committers have engaged (messages #9, #11, #14), and the synchronous pipeline path was accepted in the thread as a scope choice. Without the code I could not check that the new state has no blocking PQgetResult() left and handles connection loss.
- already-exists: 0 objection(s). No existing feature gives the same result: -M extended uses unnamed statements, and --jobs equal to --client only hides the stall by spending one thread per client. v5 uses the existing PQsendPrepare() and the existing pgbench event loop, so it adds no second mechanism.
- demand: 0 objection(s). The thread shows enough demand for a fix this local. A reproducible stall behind PgBouncer in session pooling mode is reported in message #1. Message #14 adds a second case without a pooler: a Prepare waiting on a table lock stalls unrelated clients on the same worker, which skews what pgbench measures. Running with --jobs equal to --client, or using -M extended, avoids the stall, but neither keeps -M prepared measurements valid with fewer threads than clients. The cost stays inside pgbench, with no libpq or UI change in v5.

### Precedents retrieved

- P1: [New GUC prepared_statement_limit to limit memory used by prepared statements](https://commitfest.postgresql.org/patch/2241/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P2: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P3: [cursors with prepared statements](https://commitfest.postgresql.org/patch/1667/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P4: [query logging of prepared statements](https://commitfest.postgresql.org/patch/2002/) (cf, Withdrawn, 2019-04-22) - judged not relevant; reason summarised from its thread
- P5: [Adding column "mem_usage" to view pg_prepared_statements](https://commitfest.postgresql.org/patch/2223/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P6: [Simplify standby state machine a bit in WaitForWALToBecomeAvailable()](https://commitfest.postgresql.org/patch/3965/) (cf, Withdrawn, 2023-03-03) - judged not relevant; reason summarised from its thread
- P7: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P8: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - judged not relevant; reason summarised from its thread
- P9: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P10: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P11: [Add pg_wait_for_lockers() function](https://commitfest.postgresql.org/patch/4480/) (cf, Withdrawn, 2024-07-22) - judged not relevant
- P12: [Built-in connection pooler](https://commitfest.postgresql.org/patch/3043/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant
- P14: [Move SLRU data into the regular buffer pool](https://commitfest.postgresql.org/patch/3514/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P15: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P16: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant
- P17: [parse-time support function for "any" datatype](https://commitfest.postgresql.org/patch/2081/) (cf, Withdrawn, 2021-03-09) - judged not relevant
- P18: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P19: [Lockless queue of waiters based on atomic operations for LWLock](https://commitfest.postgresql.org/patch/3983/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P20: [Synchronizing slots from primary to standby](https://commitfest.postgresql.org/patch/1961/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Support multiple synchronous standby servers](https://commitfest.postgresql.org/patch/293/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P22: [function pg_setting_value_split() to parse shared_preload_libraries etc.](https://commitfest.postgresql.org/patch/2782/) (cf, Withdrawn, 2020-11-29) - judged not relevant
- P23: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [notice lock waits](https://commitfest.postgresql.org/patch/702/) (cf, Returned with feedback, 2016-10-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 86 s; cost reported by the CLI: $0.98
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:54:53.726Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
