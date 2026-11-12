# Design pre-review: Backend crash in expanded_record_info function

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7618](https://commitfest.postgresql.org/patch/7618/) (Needs review); authors: Tender Wang (tndrwang)
- Thread: [Backend crash in expanded_record_info functions](https://www.postgresql.org/message-id/flat/CAHewXNnDBGifxSqHvt1Xg4XmTL4%3DkQauQRo2kyWP4Yhz0bPG9w%40mail.gmail.com), started 2026-11-10
- Messages read: 1 of 1
- Patch prose read: 0001-Validate-expanded-record-test-function-arguments.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. Message #1 states the problem with a reproducible crash and a backtrace, and the patch changes only three test functions in src/test/regress/regress.c. The confidence is medium because the thread has one message and nobody has read the code here.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody checked that the row-type check accepts every argument the existing tests pass, including domains over composite types. The error message is also unseen.

## The proposal, restated

**Problem.** The regression test functions expanded_record_info(), expanded_record_identity() and expanded_record_field_text() take a polymorphic argument, so a scalar such as an integer passes function resolution. They then treat the Datum as a pointer to an expanded record, so `SELECT expanded_record_info(2);` in the regression database crashes the backend. SQLsmith found the crash on current master.

**Proposed design.** Before converting the Datum, each of the three C functions checks that the resolved argument type is a row type. The change touches only src/test/regress/regress.c.

**User-visible surface.**
- expanded_record_info(), expanded_record_identity() and expanded_record_field_text() in the regression database now reject non-row arguments instead of crashing.

**What the author claims.**
- SQLsmith on current master found a query that crashes the backend.
- After the regression tests run, `SELECT expanded_record_info(2);` reproduces the crash.
- The same flaw exists in expanded_record_identity() and expanded_record_field_text().

**Thread state.** Only the first message is visible. Nobody has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-10; 24 candidates retrieved, none is the same or a similar idea).

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
| backpatchability | **yes** | The entry is tagged Bugfix, and the proposal does not say whether back branches have the same three functions and need the fix. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do the back branches contain the same three functions, and should the fix go there?
2. What error message and error code does a non-row argument produce?
3. Do other polymorphic test functions in regress.c convert their argument without a type check?
4. Does the patch add a regression test for the new check, or is the check alone intended?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the expanded-record code or the regression test support functions in regress.c would have to own it. Nobody has replied to message #1.
- **Liability:** Small: three argument checks in test-only C functions that exist only in the regression database. The owner signs for the checks not rejecting any argument the existing tests pass.
- **Author follow-up:** No sign either way; the thread has only the first message, which includes a reproducer, a backtrace and a patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 2.6 KB and touches only src/test/regress/regress.c. It helps developers and fuzzers such as SQLsmith that run queries in the regression database, and it adds no user-visible surface.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] shares only the theme of regression-test crashes: it was a withdrawn proposal to add a permanent TAP test for an already-removed Assert, while this patch adds argument checks to three test functions in regress.c and is not shown to add a test. From the prior-attempts angle there is nothing to object to. Not checked without the code: the error message, whether other regress.c functions have the same flaw, and back-branch scope.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The patch changes only C test functions in src/test/regress/regress.c, which exist only in the regression database. It adds no syntax, GUC, catalog object, pg_dump or psql surface, and no client-visible behaviour beyond replacing a crash with an error. P3 concerns adding a permanent test and does not apply to this fix.
- replication-ondisk: 0 objection(s). The patch changes only argument checks in regression test functions in src/test/regress/regress.c. It has no on-disk, WAL, replication, catalog, wire-protocol or extension-ABI state, so this angle has nothing to object to.
- security: 0 objection(s). The functions exist only in the regression database, built from regress.c, and are never installed in a production cluster. The patch adds an argument check and grants no new privilege or exposure, so there is no security angle to object to.
- maintenance-fit: 0 objection(s). The patch adds argument checks to three test-only C functions in src/test/regress/regress.c. It adds no per-query cost, no catalog or API surface, and nothing for users outside the regression database, so the maintenance cost is near zero. [P3] does not apply because that thread was about adding a permanent test, and this patch fixes a crash. The material does not show whether other functions in regress.c have the same flaw.
- already-exists: 0 objection(s). Postgres has no existing feature, contrib module or extension that stops these regress.c test functions from crashing on a scalar argument. Narrowing the SQL declaration to a row type would change test behaviour, and the proposal uses the usual type check, so it adds no duplicate mechanism. [P3] is about adding a test and does not apply to a crash fix.
- demand: 0 objection(s). The demand angle has no objection. The users are developers and fuzzers such as SQLsmith that run in the regression database, and message #1 shows a reproducible crash. The fix changes only src/test/regress/regress.c, so it adds nothing to user surface, catalogs or hot paths. [P3] is about adding a permanent test for a removed Assert, so its reason does not apply to a crash fix in test code.

### Precedents retrieved

- P1: [Mark function arguments of type "T *" as "const T *" where possible](https://commitfest.postgresql.org/patch/6302/) (cf, Withdrawn, 2026-01-13) - judged not relevant; reason summarised from its thread
- P2: [calculate and use most common type for polymorphic arguments](https://commitfest.postgresql.org/patch/258/) (cf, Rejected, 2015-07-22) - judged not relevant; reason summarised from its thread
- P3: [Test for function error in logrep worker](https://commitfest.postgresql.org/patch/4046/) (cf, Withdrawn, 2023-04-05) - used in the review; reason summarised from its thread
- P4: [change regexp_substr first argument make tests more easier to understand](https://commitfest.postgresql.org/patch/4744/) (cf, Withdrawn, 2024-09-23) - judged not relevant; reason summarised from its thread
- P5: [allow mix of composite and scalar variables in target list](https://commitfest.postgresql.org/patch/1139/) (cf, Returned with feedback, 2017-10-30) - judged not relevant; reason summarised from its thread
- P6: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant; reason summarised from its thread
- P7: [Add test module for verifying backtrace functionality](https://commitfest.postgresql.org/patch/4823/) (cf, Withdrawn, 2024-08-24) - judged not relevant; reason summarised from its thread
- P8: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P9: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [postgres_fdw - make cached connection functions tests meaningful with debug_invalidate_system_caches_always](https://commitfest.postgresql.org/patch/3112/) (cf, Rejected, 2021-07-21) - judged not relevant
- P11: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P12: [Revert addition of third argument to format_type().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7564654adf07ec26b83c7effc7f54f7183e04519) (revert, Reverted, 2011-03-10) - judged not relevant
- P13: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant
- P14: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant
- P15: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant
- P16: [Support prepared statement invalidation when result or argument types change](https://commitfest.postgresql.org/patch/4518/) (cf, Withdrawn, 2024-07-25) - judged not relevant
- P17: [polymorphic table functions](https://commitfest.postgresql.org/patch/2405/) (cf, Returned with feedback, 2020-03-30) - judged not relevant
- P18: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P19: [Revert badly-broken patch to dump comments on composite-type columns.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=18661f282bae5599abb68f53bc8b451b2d67bd75) (revert, Reverted, 2004-08-06) - judged not relevant
- P20: [Revert the logic for expanding AND/OR conditions in pred_test() to what it was in 7.4, and add some comments explaining why it has to be this way.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e89bae704f79af8b2b6ac3d43ea963c22df1947) (revert, Reverted, 2005-03-01) - judged not relevant
- P21: [Expand the use of check_canonical_path() for more GUCs](https://commitfest.postgresql.org/patch/2564/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P22: [Fix bug when calling strncmp in check_authmethod_valid](https://commitfest.postgresql.org/patch/4960/) (cf, Withdrawn, 2024-04-30) - judged not relevant
- P23: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 81 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 8 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 41 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:08:07.095Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
