# Design pre-review: Fix leak when a plpgsql exception block catches an error from CALL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7432](https://commitfest.postgresql.org/patch/7432/) (Needs review); authors: Muzzammil Sarwar (muzzammil242)
- Thread: [\[PATCH\] Fix leak when a plpgsql exception block catches an error from CALL](https://www.postgresql.org/message-id/flat/CABUcsRSoj8i475YkPtVFETbaOvSzA4SnAwPRLr5bJKF3gieASQ%40mail.gmail.com), started 2026-10-11
- Messages read: 1 of 1
- Patch prose read: v1-0001-Fix-leak-when-a-plpgsql-exception-block-catches-a.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1a or 1b: message #1 gives a reproducible leak, its cause and measurements. The only surviving objection, maintenance-fit-1, is a small fixable design point at level 2. The fix changes which resource owner holds a plan pin and is meant for back-patch to 14 through 19, so a committer who knows the PL/pgSQL and SPI code has to sign for it. Nobody has replied yet.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Not checked: whether every path that can end the transaction inside a CALL or DO is covered by SPI_inside_nonatomic_context(), and whether the new test is stable on the buildfarm.

## The proposal, restated

**Problem.** A PL/pgSQL procedure that catches failing CALLs in an exception block keeps each failed CALL's plan until it returns, about 2 kB per failure. Batch procedures that log failed rows and go on can use memory without limit, about 2 GiB per million failures. DO blocks leak the same way, also in atomic contexts.

**Proposed design.** exec_stmt_call() passes the procedure-lifespan resource owner to SPI only when SPI_inside_nonatomic_context() is true, that is, when the called procedure could end the transaction. Otherwise it leaves the owner NULL, so SPI uses the current resource owner, and aborting the exception block's subtransaction releases the pin. The patch also adds a regression test that counts CachedPlan contexts.

**What the author claims.**
- The leak exists on master and on 14 through 19, as an oversight in commit ee895a655.
- 20,000 caught failures leave 20,005 extra CachedPlan contexts without the patch and 5 with it, and memory stays flat with or without COMMIT.
- The extra test per CALL costs nothing measurable: 3.07s against 3.01s for 1,000,000 empty CALLs, within noise.
- A per-CALL child resource owner also fixes the leak but costs about 3-4% per CALL.
- The patch cherry-picks cleanly to 14 through 19 and should be back-patched, as a6b1f5365d5 was.

**Thread state.** Only the first message is visible. Nobody has replied yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-11; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Correctness depends on two copies of the non-atomic rule agreeing

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch changes when _SPI_execute_plan() runs a CALL non-atomically but leaves SPI_inside_nonatomic_context() unchanged. A CALL then runs non-atomically while its plan is pinned in the transaction's resource owner, and a COMMIT in the callee releases the pin while the plan is still in use. Users would hit this as a crash or a refcount warning on an uncommon path, long after the change was committed.

Message #1 says SPI_inside_nonatomic_context() "also implements" the rule from c96de42c4b5, which reads as a second copy of the rule that _SPI_execute_plan() applies, not a shared helper. After this patch, pl_exec.c relies on both copies giving the same answer to decide which resource owner holds the pin. The proposal does not say what fails loudly, such as an assertion, if the two copies drift apart. This is a small point for a back-patched bug fix, but a committer would likely ask for it before signing for the change on six branches.

Thread: [#1 (Muzzammil Sarwar, 2026-10-11)](https://www.postgresql.org/message-id/CABUcsRSoj8i475YkPtVFETbaOvSzA4SnAwPRLr5bJKF3gieASQ%40mail.gmail.com)

**Question:** Do exec_stmt_call() and _SPI_execute_plan() share one test for non-atomic execution, or can an assertion catch a CALL that runs non-atomically while its pin is in the current resource owner?

Counter-check: held. spi.c does hold the rule twice: in _SPI_execute_plan() and in SPI_inside_nonatomic_context(). Drift would likely show as a plancache reference leak warning in the existing transaction tests, but the proposal does not say so, so the request for a shared test or assertion stands as a small point.

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
| backpatchability | **yes** | The author asks for a back-patch to 14 through 19 and adds a new regression test on each branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | exec_stmt_call() now depends on SPI_inside_nonatomic_context() agreeing with the non-atomic test in _SPI_execute_plan(). |
| performance on the hot path | **yes** | The patch adds one test to every PL/pgSQL CALL; message #1 measures it as within noise. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do exec_stmt_call() and _SPI_execute_plan() share one test for non-atomic execution, or can an assertion catch a CALL that runs non-atomically with its pin in the current resource owner?
2. Which existing test fails if SPI_inside_nonatomic_context() and _SPI_execute_plan() disagree about whether a CALL runs non-atomically?
3. Does the test that counts CachedPlan contexts in pg_backend_memory_contexts give stable output on all six branches under parallel schedules and unusual build options?
4. Do the TAP tests pass with the patch on master and on the back branches?
5. Does any other PL or extension depend on an atomic DO block pinning its CALL plans in the block's simple-expression resource owner?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the PL/pgSQL executor, SPI plan execution and the plan cache's resource owner handling; none has replied in the visible thread.
- **Liability:** The owner signs for the plan pin lifetime of every CALL and DO in PL/pgSQL on master and on five stable branches, and for a new test that reads pg_backend_memory_contexts on all of them.
- **Author follow-up:** No reply to test yet; the author tested six branches, measured an alternative design and offered to drop the test if it is thought fragile.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 7.8 KB, touches only pl_exec.c and the plpgsql_call test, and adds no user-visible surface. It fixes unbounded memory growth of about 2 kB per caught CALL failure in batch procedures.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P12] and [P14] free CachedPlan contexts on DROP FUNCTION, a different leak with a different fix, and their recorded reason (other backends keep their plans) does not apply to a per-backend resource owner choice in exec_stmt_call(). Message #1 cites ee895a655 as the cause and a6b1f5365d5 as a back-patched fix for an earlier CALL leak, but neither is a rejected attempt at this fix.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch adds no syntax, GUC, catalog object or psql change, and it changes no dump output. The only behaviour change is which resource owner holds a plan pin during a CALL or DO. A CALL that cannot end the transaction now uses the current resource owner, as ee895a655 describes for atomic contexts, so the back-patch does not change any result a user can observe. Two points could not be checked without the code: whether the new test that reads pg_backend_memory_contexts behaves the same on every back branch, and whether any extension depends on the old resource owner choice for atomic DO blocks.
- replication-ondisk: 0 objection(s). The patch changes only which in-memory resource owner holds a plan pin during CALL and DO in pl_exec.c. It adds no on-disk, WAL, catalog, wire protocol or exported struct change, so pg_upgrade, standbys, subscribers and the back-branch ABI are not affected. [P12] and [P14] dealt with freeing plans on DROP across sessions, a different problem.
- security: 0 objection(s). The patch has no privilege or exposure aspect. It only changes which resource owner holds a plan pin for CALL and DO inside PL/pgSQL, adds no new object, role, setting or view, and lets no one see or do more than before. The leak it fixes affects only the session that runs the loop, and the patch removes that growth.
- maintenance-fit: 1 objection(s). Apart from the point above, the fix looks like a fit for core: it is a narrow bug fix in pl_exec.c, it adds no new mechanism or API, and the author measured the cost per CALL. The precedents [P12] and [P14] concern freeing plans on DROP and do not apply here.
- already-exists: 0 objection(s). This is a bug fix for a memory leak, and no existing feature, setting or extension avoids the leak. The only workaround, force_generic_plan, still leaves a pin and a resource owner entry behind for each failure (message #1). The fix reuses existing SPI behaviour and SPI_inside_nonatomic_context(), so it adds no second mechanism. Precedents P12 and P14 cover cached plans freed on DROP FUNCTION, a different leak.
- demand: 0 objection(s). This is a bug fix for unbounded memory growth, and from the demand angle it should stand without field reports. Message #1 shows a reproducible leak of about 2 kB per caught CALL failure in a common batch pattern, and the cost falls on no one else, with no new UI and no measured per-CALL slowdown. [P12] and [P14] dealt with freeing plans at DROP time, and their reason does not apply here.

### Precedents retrieved

- P1: [Avoid use of TopMemoryContext for resource owner cleanup in portals](https://commitfest.postgresql.org/patch/6603/) (cf, Returned with feedback, 2026-07-07) - judged not relevant; reason summarised from its thread
- P2: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/2834/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P4: [ResourceOwner refactoring](https://commitfest.postgresql.org/patch/3364/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P5: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/1361/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P6: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/2911/) (cf, Withdrawn, 2023-10-30) - judged not relevant; reason summarised from its thread
- P7: [Revert "Close stdin where it's not needed in TestLib.pm procedures"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca266a069a20c32a8f0a1df982a5a67d9483bcb3) (revert, Reverted, 2019-11-27) - judged not relevant
- P8: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant; reason summarised from its thread
- P9: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P10: [Revert: Implement pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=772faafca1b288c4dd66b7150a7831c27b768003) (revert, Reverted, 2024-04-11) - judged not relevant
- P11: [Leaking memory in text_overlay function](https://commitfest.postgresql.org/patch/658/) (cf, Rejected, 2016-08-04) - judged not relevant; reason summarised from its thread
- P12: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5974/) (cf, Withdrawn, 2025-08-19) - used in the review; reason summarised from its thread
- P13: [pg_upgrade: fix memory leak in SLRU I/O code](https://commitfest.postgresql.org/patch/6462/) (cf, Withdrawn, 2026-02-05) - judged not relevant
- P14: [When deleting the plpgsql function, release the CachedPlan of the function](https://commitfest.postgresql.org/patch/5975/) (cf, Withdrawn, 2025-12-31) - used in the review
- P15: [The Prepare statement with the skewed parameter uses the custom plan](https://commitfest.postgresql.org/patch/4839/) (cf, Withdrawn, 2024-09-22) - judged not relevant
- P16: [Revert thinko introduced into prefix_selectivity() by my recent patch: make_greater_string needs the < procedure not the >= one. Spotted by Peter.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=164899db1ceed1e582e5ae1af9455740e7aaa94a) (revert, Reverted, 2008-03-17) - judged not relevant
- P17: [Free memory allocated by waitonlock_error_callback()](https://commitfest.postgresql.org/patch/6069/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P18: [Implicit prepare of statements: replace literals with parameters and store cached plans](https://commitfest.postgresql.org/patch/1150/) (cf, Returned with feedback, 2018-03-06) - judged not relevant
- P19: [Revert "Refactor ChangeVarNodesExtended() using the custom callback"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2782f3b8454676c9df408aa0bc960efca0f3eb22) (revert, Reverted, 2025-05-03) - judged not relevant
- P20: [Revert: Avoid looping over all type cache entries in TypeCacheRelCallback()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8daa62a10c911c851f7e9ec5ef7b90cfd4b73212) (revert, Reverted, 2024-08-25) - judged not relevant
- P21: [Revert inappropriate weakening of an Assert in plpgsql.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=16a3ae504e2030583851c8f45cbed14b0c546d6e) (revert, Reverted, 2025-03-21) - judged not relevant
- P22: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant
- P23: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P24: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 96 s; cost reported by the CLI: $1.01
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:55:04.514Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
