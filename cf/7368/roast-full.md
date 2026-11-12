# Design pre-review: Two fixes for parallel query cleanup: DSM detach order and a stray statement timeout

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7368](https://commitfest.postgresql.org/patch/7368/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [Two fixes for parallel query cleanup: DSM detach order and a stray statement timeout](https://www.postgresql.org/message-id/flat/CAGRkXqQ7d%2BB%2BXDCNR0oUmG55EJ421HPj3saWONU2Uc0VKqARWg%40mail.gmail.com), started 2026-09-30
- Messages read: 1 of 1
- Patch prose read: v1-0001-Detach-the-parallel-DSM-segment-after-the-workers.patch, v1-0002-Forget-a-statement-timeout-that-fires-after-the-s.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived the panel, so nothing stops either patch at levels 1a, 1b or 2. Both are bug fixes with measured reproducers, and 0002 restores the behaviour before 22f6f2c1ccb. Confidence is low because only message #1 is visible, no one has replied, and two safety points need the code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the panel could not check whether 0002 clears only the timeout's own cancel request, or whether 0001 keeps the leader safe when a worker never exits. The pass at levels 1 and 2 is provisional on both points.

## The proposal, restated

**Problem.** A user who cancels a parallel query gets errors in the log from workers that start after the leader detached the DSM segment. A statement_timeout that fires during end-of-query cleanup, such as temp file deletion in dsm_detach() with interrupts held, stays pending and cancels the next statement, even a COMMIT.

**Proposed design.** 0001 moves dsm_detach() in DestroyParallelContext() to after WaitForParallelWorkersToExit(), but still detaches the error queues before the wait. 0002 changes disable_statement_timeout() so it clears the fired indicator and the queued cancel request when the timer has fired but was not serviced. The patches are independent.

**User-visible surface.**
- Cancelled parallel queries no longer log errors from workers that fail to map the segment or attach to the DSA or SharedFileSet.
- A statement_timeout that fires after a statement's last CHECK_FOR_INTERRUPTS() no longer cancels the next statement.

**What the author claims.**
- Cancelling a 4-worker parallel hash join 1200 times logged 407 errors on master and none with 0001.
- With 0001, cancelling workers blocked on full tuple queues, parallel CREATE INDEX and parallel VACUUM showed no hang and no leftover temp files.
- Without 0002, a follow-up statement failed 5 of 120 times, and COMMIT failed 4 of 242 times, leaving an aborted transaction block; with 0002 both counts drop to zero.
- 0002 fixes a regression from 22f6f2c1ccb present in 13 and later, and restores the earlier behaviour; Tom described the case in [2].
- The REPACK decoding worker copied the same shutdown sequence, and its early dsm_detach() was fixed separately.

**Thread state.** Only the first message is visible. No one has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-30; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | Message #1 calls 0002 a regression in 13 and later but does not say which branches either patch targets. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Both patches change cleanup ordering and interrupt bookkeeping in parallel.c and postgres.c, where timing bugs are hard to reproduce. |
| performance on the hot path | **yes** | 0002 changes disable_statement_timeout(), which runs once per statement; no added cost is claimed or shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can 0002 clear a cancel request from a client cancel or pg_cancel_backend() that arrives in the same window as the fired statement timeout?
2. With 0001, what does the leader do with the DSM segment if a worker never exits or the postmaster dies during WaitForParallelWorkersToExit()?
3. Do lock_timeout, transaction_timeout or the idle timeouts have the same fired-but-unserviced case, and does 0002 cover them?
4. Which back branches is each patch meant for, given that 0002 is described as a regression in 13 and later?
5. How does the attached reproducer set up the timing, so that a reviewer can repeat the 5 in 120 and 4 in 242 failure counts?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. 0001 needs someone who knows parallel context shutdown and DSM lifetime. 0002 needs someone who knows the timeout and interrupt handling in postgres.c.
- **Liability:** The committer signs for cleanup ordering in every parallel query and for cancel handling at the end of every statement. If 0002 is back-patched, that covers all supported branches from 13.
- **Author follow-up:** No sign either way; only message #1 exists. The author ran stress tests and attached a reproducer.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** 0002 fixes a case where a COMMIT fails and leaves an aborted transaction block, for users who combine statement_timeout with parallel queries. Both patches are small and add no user-visible surface, but the timing reasoning needs a reviewer who knows these two areas.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches either patch. [P2] also touches statement_timeout, but it changed which cancel message is reported and fixed a flaky test, and its thread shows no objection that would apply here. Message #1 points to two earlier discussions: [1], where the early dsm_detach() in the REPACK decoding worker was fixed, and [2], where Tom is said to have described the stray-timeout case; the material does not show what either thread decided.
- standards-compat: 0 objection(s). Neither patch changes SQL syntax, GUCs, catalogs, pg_dump output, psql or the wire protocol. 0002 changes visible behaviour only by stopping a spurious statement_timeout error on the next statement, which matches the behaviour before 22f6f2c1ccb. The material does not say whether 0002 can also discard a client cancel request that arrives in the same window. That case is already racy under the current cancel protocol, so it is better left to the safety angle than raised here as a compatibility objection.
- replication-ondisk: 0 objection(s). Neither patch touches persistent or cross-node state: no on-disk format, WAL record, catalog, wire protocol message or exported struct changes, so pg_upgrade, standbys and back-branch ABI are unaffected. The DSM segment and SharedFileSet temp files are per-query and per-process; the open cleanup question on worker hang or postmaster death needs the code to judge.
- security: 0 objection(s). Neither patch changes who may do what or what anyone can see. 0001 reorders dsm_detach() after WaitForParallelWorkersToExit() inside the leader's own parallel context, and 0002 changes timeout bookkeeping inside one backend. Neither adds a privilege, a role check, a new exposure path or a resource an unprivileged user can grow without bound. One question belongs to the correctness angle: whether 0002 can drop a pg_cancel_backend() request that arrives in the same window. If that happens, an administrator's cancel would be lost. No one gains access or privileges they did not already have.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle, neither patch adds a GUC, catalog state, API, dependency or hot-path work. 0001 reorders existing cleanup in one function. 0002 restores how disable_statement_timeout() behaved before 22f6f2c1ccb, and it runs only once per statement. Both are core bug fixes with nothing that could live in an extension. Two points could not be checked without the code and belong to other angles. One is whether 0002 can clear a user cancel that arrives in the same window. The other is whether lock_timeout or transaction_timeout need the same change. Committability: no committer has engaged yet. [P2] touches statement_timeout but records no reason that applies here.
- already-exists: 0 objection(s). Both patches are bug fixes in core code, parallel.c and the statement timeout handling in postgres.c. No existing feature, setting, extension or hook avoids the stray timeout or the worker attach errors, so nothing from this angle applies. The only precedent, [P2], concerns a different timeout error message and was withdrawn without a recorded objection.
- demand: 0 objection(s). Both patches fix bugs with measured reproducers, so the demand test does not apply in the usual way. 0002 fixes a regression present since 13 that can abort a COMMIT; 0001 removes errors logged after an ordinary cancel; neither adds UI or cost for non-users.

### Precedents retrieved

- P1: [Cancel problems of query to pg_stat_statements](https://commitfest.postgresql.org/patch/5825/) (cf, Rejected, 2026-03-25) - judged not relevant; reason summarised from its thread
- P2: [Fix error message for statement_timeout and fix flaky test](https://commitfest.postgresql.org/patch/5072/) (cf, Withdrawn, 2024-06-25) - used in the review; reason summarised from its thread
- P3: [timeout parameters in libpq](https://commitfest.postgresql.org/patch/1843/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P4: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P5: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P6: [Logical replication timeout](https://commitfest.postgresql.org/patch/5601/) (cf, Returned with feedback, 2025-09-01) - judged not relevant; reason summarised from its thread
- P7: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant; reason summarised from its thread
- P8: [minor fix in CancelVirtualTransaction](https://commitfest.postgresql.org/patch/1903/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P9: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - judged not relevant
- P10: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P11: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P12: [Microsecond-based timeouts](https://commitfest.postgresql.org/patch/4245/) (cf, Returned with feedback, 2023-07-26) - judged not relevant
- P13: [hostorder and failover_timeout for libpq](https://commitfest.postgresql.org/patch/1631/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P14: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P15: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P16: [Assert that locks are properly held in PageGetLSN](https://commitfest.postgresql.org/patch/1279/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P17: [Revert b6002a796](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28b3e3905c982c42fb10ee800e6f881e9742c89d) (revert, Reverted, 2021-04-01) - judged not relevant
- P18: [pgrename_temp() – concurrent rename of temporary files on Windows](https://commitfest.postgresql.org/patch/2230/) (cf, Returned with feedback, 2020-04-05) - judged not relevant
- P19: [Temporary file access API](https://commitfest.postgresql.org/patch/3722/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P21: [remove spurious CREATE INDEX CONCURRENTLY wait](https://commitfest.postgresql.org/patch/2720/) (cf, Withdrawn, 2020-09-01) - judged not relevant
- P22: [Disable parallel query by default](https://commitfest.postgresql.org/patch/5751/) (cf, Rejected, 2026-02-11) - judged not relevant
- P23: [Change EXPLAIN's ANALYZE option to the clearer (EXEC)](https://commitfest.postgresql.org/patch/2121/) (cf, Withdrawn, 2019-07-02) - judged not relevant
- P24: [temporary file usage report with extended protocol and unnamed portals](https://commitfest.postgresql.org/patch/5711/) (cf, Withdrawn, 2025-08-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 66 s; cost reported by the CLI: $0.52
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 45 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:46:30.582Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
