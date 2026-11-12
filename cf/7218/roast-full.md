# Design pre-review: Assorted Win32 error handling fixes (CreateThread, pgwin32_select, pg_usleep)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7218](https://commitfest.postgresql.org/patch/7218/) (Needs review); authors: Sehrope Sarkuni (sehrope)
- Thread: [Assorted Win32 error handling fixes (CreateThread, pgwin32_select, pg_usleep)](https://www.postgresql.org/message-id/flat/CAH7T-apWWfEt%2BV8uLU7KXv7CMM9LoqJ1Zo2LdWmXMz%3Du-Ek9%3Dw%40mail.gmail.com), started 2026-06-10
- Messages read: 3 of 3
- Patch prose read: v2-0003-Fix-off-by-one-in-pgwin32_select-event-array.patch, v2-0004-Do-not-treat-a-failed-wait-as-a-timeout-in-win32-.patch, v2-0001-Fix-CreateThread-failure-check-in-win32-timer-cod.patch, v2-0002-Fix-CreateThread-failure-check-in-pg_test_fsync.patch, 0001-Fix-CreateThread-failure-check-in-win32-timer-code.patch, 0002-Fix-CreateThread-failure-check-in-pg_test_fsync.patch, 0003-Fix-off-by-one-in-pgwin32_select-event-array.patch, 0004-Do-not-treat-a-failed-wait-as-a-timeout-in-win32-pg_.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived the panel, so nothing stops patches 0003 and 0004 at levels 1a, 1b or 2. Both are small, local fixes to Win32 port code with a stated problem and no new user-facing surface. Confidence is low because nobody has reviewed 0004, and the thread does not say whether elog(FATAL) is safe for every pg_usleep() caller.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, so the 0003 array arithmetic and the exact 0004 control flow are unchecked. Whether the new FATAL can fire in a context where elog() is unsafe needs the code and the list of pg_usleep() callers.

## The proposal, restated

**Problem.** Several Win32 port functions mishandle error returns. The author found them with new analysis tooling. The two CreateThread fixes are already committed, so the open problems are a one-element overflow in the pgwin32_select() events array and pg_usleep() treating WAIT_FAILED as a timeout, which can make retry loops spin.

**Proposed design.** Patch 0003 sizes the pgwin32_select() events array as 2*FD_SETSIZE + 1 so the signal event fits after full, disjoint read and write fd sets. Patch 0004 makes win32 pg_usleep() handle WAIT_OBJECT_0 and WAIT_TIMEOUT explicitly and call elog(FATAL) on any other result from WaitForSingleObject().

**User-visible surface.**
- pg_usleep() on Windows now raises FATAL when WaitForSingleObject() fails, where it used to return at once.

**What the author claims.**
- 0003 and 0004 are defensive fixes with no reachable in-core trigger.
- 0004 changes behavior on the failure path and needs review; 0003 is mechanical.
- The series passes the author's GitHub Actions CI run.
- Only 0001, now committed, had practical server impact.

**Thread state.** Heikki Linnakangas committed 0001 and 0002 on 2026-10-15 and said he did not look at 0003 and 0004. Nobody has reviewed 0004 yet, and CFbot reports that the patch needs a rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-10; 24 candidates retrieved, none is the same or a similar idea).

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
| security / privileges | **yes** | Patch 0003 removes a one-element write past the end of the events array in pgwin32_select(). |
| backpatchability | **yes** | The entry is tagged Bugfix, and the thread does not say whether 0003 and 0004 should be back-patched. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is elog(FATAL) safe for every caller of win32 pg_usleep(), including the postmaster and code that runs in a critical section?
2. Can pg_usleep() run before error reporting is set up, or from inside error reporting, so that the new elog(FATAL) fails or recurses?
3. Why is FATAL the right response to WAIT_FAILED, compared with a log message followed by a plain Sleep() for the requested time?
4. Should 0003 and 0004 be back-patched, given that the author says neither has a reachable in-core trigger?
5. Can pgwin32_select() receive full, disjoint read and write fd sets from any extension or frontend caller, or is 0003 defensive only?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Heikki Linnakangas committed 0001 and 0002 in message #3 and said he did not look at 0003 and 0004. They need a committer who knows the Win32 signal emulation and the pg_usleep() callers.
- **Liability:** The committer signs for a new FATAL exit in win32 pg_usleep(), which many backend and postmaster paths call. The changed lines are few, but the set of affected callers is large and Windows-only.
- **Author follow-up:** The author posted a rebased v2 in message #2 and moved the entry to the next CommitFest. CFbot has reported a needed rebase since 2026-10-15, with no reply in the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The two remaining patches are about 1.3 KB each, touch only src/backend/port/win32, and by the author's account in message #2 have no reachable in-core trigger. 0003 is mechanical; 0004 needs a rebase and a Windows-aware reviewer for the FATAL choice.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier proposal with the same or a similar idea. The thread does not point to any earlier discussion of the pgwin32_select() array size or of WAIT_FAILED handling in pg_usleep(), so the prior-attempts angle has nothing to raise.
- standards-compat: 0 objection(s). Neither remaining patch changes SQL syntax, GUCs, catalogs, pg_dump output, psql or the wire protocol. The one behavior change, FATAL instead of an immediate return when WaitForSingleObject() fails inside pg_usleep(), is on an internal failure path that the author says no in-core code can reach. Whether FATAL is safe for every caller is a safety question for another angle.
- replication-ondisk: 0 objection(s). The remaining patches 0003 and 0004 touch only process-local Win32 code: an in-memory event array in pgwin32_select() and the error path of pg_usleep(). Neither changes on-disk format, WAL, replication, catalogs, the wire protocol or an exported struct or signature, so this angle finds nothing to object to.
- security: 0 objection(s). Neither remaining patch changes who can do what or what anyone can learn, and no unprivileged user can trigger the failure paths. 0003 removes a one-element overflow on the stack, which helps safety, and the FATAL in 0004 fires only on an internal handle failure that a client cannot cause.
- maintenance-fit: 0 objection(s). Both remaining patches are small, local fixes in src/backend/port/win32 with no new API, GUC, catalog or hot-path cost, so there is no maintenance or fit objection to raise. Whether elog(FATAL) is safe for every pg_usleep() caller is a safety question for another angle. Committability depends on a Windows-aware committer picking up 0004, since Heikki did not look at it in message #3.
- already-exists: 0 objection(s). Both remaining patches are bug fixes to internal Win32 port code, so no existing feature, extension or external tool can serve in their place. Patch 0003 corrects an array size in pgwin32_select(), and patch 0004 changes how win32 pg_usleep() handles a failed WaitForSingleObject() call. Neither duplicates an existing mechanism or adds a second way to do one job.
- demand: 0 objection(s). Demand is the wrong test for these fixes. 0003 fixes a one-element array overrun and 0004 adds a missing error check, so neither adds a feature or user-facing surface that needs users; the author's "no reachable in-core trigger" bears on priority and back-patching. Whether FATAL is safe for every pg_usleep() caller is a design question for another angle.

### Precedents retrieved

- P1: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P2: [pg_usleep for multisecond delays](https://commitfest.postgresql.org/patch/4175/) (cf, Returned with feedback, 2023-11-07) - judged not relevant; reason summarised from its thread
- P3: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant; reason summarised from its thread
- P4: [Revert "Fix link error for test_radixtree module on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6d9751fa8fd40c988541c9c72ac7a2095ba73c19) (revert, Reverted, 2024-03-08) - judged not relevant
- P5: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P6: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P7: [Use WaitLatch for {pre, post}_auth_delay instead of pg_usleep](https://commitfest.postgresql.org/patch/3255/) (cf, Rejected, 2021-07-29) - judged not relevant; reason summarised from its thread
- P8: [Eliminating "Permission denied" errors on stat() (on Windows)](https://commitfest.postgresql.org/patch/3058/) (cf, Withdrawn, 2021-12-03) - judged not relevant; reason summarised from its thread
- P9: [Get rid of an extra error for client disconnection on Windows](https://commitfest.postgresql.org/patch/659/) (cf, Rejected, 2016-09-13) - judged not relevant; reason summarised from its thread
- P10: [Use latch instead of pg_usleep for standby recovery conflicts](https://commitfest.postgresql.org/patch/977/) (cf, Returned with feedback, 2017-04-03) - judged not relevant; reason summarised from its thread
- P11: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P12: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P13: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P14: [Free memory allocated by waitonlock_error_callback()](https://commitfest.postgresql.org/patch/6069/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P15: [Remove investigative code for can't-reattach-to-shared-memory errors.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bcbf2346d69f6006f126044864dd9383d50d87b4) (revert, Partially reverted, 2018-05-01) - judged not relevant
- P16: [Add new wait event to XactLockTableWait](https://commitfest.postgresql.org/patch/5804/) (cf, Withdrawn, 2026-03-31) - judged not relevant
- P17: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P18: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P19: [Revert "Cannot use WL_SOCKET_WRITEABLE without WL_SOCKET_READABLE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a58db3aa10e62e4228aa409ba006014fa07a8ca2) (revert, Reverted, 2020-12-15) - judged not relevant
- P20: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant
- P21: [Make all Perl warnings fatal](https://commitfest.postgresql.org/patch/4501/) (cf, Returned with feedback, 2023-09-12) - judged not relevant
- P22: [socket_timeout in interfaces/libpq](https://commitfest.postgresql.org/patch/2175/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P23: [Improve overflow checks potentially optimized away by compiler](https://commitfest.postgresql.org/patch/400/) (cf, Returned with feedback, 2015-12-01) - judged not relevant
- P24: [new plpgsql.extra_errors check - strict_expr_check](https://commitfest.postgresql.org/patch/5044/) (cf, Withdrawn, 2026-03-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 108 s; cost reported by the CLI: $1.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:29:56.807Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
