# Design pre-review: Add a permission check to pg_stat_get_backend_subxact()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7273](https://commitfest.postgresql.org/patch/7273/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [Add a permission check to pg_stat_get_backend_subxact()](https://www.postgresql.org/message-id/flat/CAGRkXqTBZ%2BzbVuDC8xGEB6Btj61hsui5H5nGqzFBDyOXc%3D4bjQ%40mail.gmail.com), started 2026-09-08
- Messages read: 27 of 27
- Patch prose read: v12-0001-Make-per-backend-statistics-functions-respect-st.patch, 0001-Make-pg_stat_get_backend_subxact-respect-statistics-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because v12 keeps a comment on pgstat_fetch_stat_backend_by_pid() about data from an older backend. The patch applies the existing HAS_PGSTAT_PERMISSIONS() rule to four functions, on HEAD only, after pgsql-security asked for a consistency fix and a committer agreed in message #3. Confidence is medium because only 3 of 27 messages and no code were read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the handling of the stored role OID across stats reset, stats file reload and proc number reuse could not be checked. Whether the stats file format needs a version bump is also open.

## The proposal, restated

**Problem.** pg_stat_get_backend_subxact(), pg_stat_get_backend_io(), pg_stat_get_backend_wal() and pg_stat_get_backend_lock() return a session's details to any caller, unlike their sibling functions that call HAS_PGSTAT_PERMISSIONS(). The author reported it to pgsql-security, which judged it not a vulnerability and asked for a consistency fix on HEAD.

**Proposed design.** The four functions call HAS_PGSTAT_PERMISSIONS() and hide their output from callers who are not superuser, do not have pg_read_all_stats, and are not members of the owning role. PgStat_Backend stores the owning role OID at entry creation, and the io, wal and lock functions check against it, because a stats snapshot may hold data of an older backend with the same proc number.

**User-visible surface.**
- pg_stat_get_backend_subxact() returns NULL to callers without statistics privileges.
- pg_stat_get_backend_io() and pg_stat_get_backend_lock() return no rows to such callers.
- pg_stat_get_backend_wal() returns NULL to such callers.
- Stats of processes owned by no role, such as autovacuum workers and the WAL writer, become visible only to superusers and pg_read_all_stats.
- PgStat_Backend gains a role OID field, which changes the struct layout seen by C code.

**What the author claims.**
- The change makes these functions consistent with the other per-backend functions and with pg_stat_activity.
- pid, dbid, userid and idset leak nothing new, since pg_stat_get_activity() already shows them to every caller.
- Checking the role stored in the stats entry closes the snapshot case where an unprivileged role read a superuser session's IO and WAL numbers.
- The change alters output for unprivileged callers, so it is for HEAD only, with no backpatch.

**Thread state.** Michael Paquier committed the doc fix for subxact_overflowed, and the PID-in-entry and ProcNumberGetTransactionIds() patches were dropped at his request. solai v tested v12 and found no issues, and no committer has said v12 is ready.

## 2. Prior attempts

### Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views (related)

- **Link:** [https://commitfest.postgresql.org/patch/4999/](https://commitfest.postgresql.org/patch/4999/)
- **Thread:** [https://www.postgresql.org/message-id/flat/3a60be45e7a89f50d166dba49553950d6b8a97f5.camel@cybertec.at](https://www.postgresql.org/message-id/flat/3a60be45e7a89f50d166dba49553950d6b8a97f5.camel@cybertec.at)
- **Date:** decided 2024-09-19; started 2024-05-06
- **Outcome:** Rejected (cf)
- **Who objected:** Dean Rasheed (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Dean Rasheed noted that the issue is not a security bug and that row counts can also be seen through pg_stat_get_tuples_returned(), pg_class.reltuples and plain EXPLAIN estimates, so EXPLAIN should stay as it is. After a discussion on pgsql-security, the author marked the patch rejected. The reason was that a determined attacker who can run arbitrary SQL can subvert row-level security anyway, so the restriction makes performance analysis harder without a real security gain.
- **Reason (quote):** "Therefore, restricting EXPLAIN (ANALYZE) will do more harm than good, since it will make analyzing query performance harder without a real security gain." ([message](https://www.postgresql.org/message-id/ba677c20e02f554afb5a9aef8307cf4012eb4e63.camel%40cybertec.at))
- **What is different this time:** P9 blocked a whole command for every unprivileged user. This patch applies the existing HAS_PGSTAT_PERMISSIONS() rule, so session owners and pg_read_all_stats members still see the numbers, as in pg_stat_activity.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | PgStat_Backend gains a role OID field; whether the stats file format needs a version bump can only be checked in the code. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The new role OID field changes the layout of PgStat_Backend in pgstat.h, which C extensions can see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Four functions start to hide session details from callers without pg_read_all_stats, superuser, or membership in the owning role. |
| backpatchability | **yes** | The output change for unprivileged callers is HEAD only; only the subxact_overflowed doc fix was backpatched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The subxact function checks the backend status entry role, while io, wal and lock check the role stored in PgStat_Backend. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which role OID does a PgStat_Backend entry hold after a stats reset, and for an entry read from a stats file written before the upgrade?
2. Do known monitoring tools call these four functions as a role without pg_read_all_stats, and does the change need a release note entry for them?
3. Is the race in pgstat_read_current_status() between the userid and the subxact counters accepted as existing behaviour, or is a follow-up planned?
4. Is a follow-up planned for snapshot reuse across other stats kinds, such as replication slot stats?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier takes part in the thread, committed the doc fix and steered the design; the owner needs to know the cumulative statistics and backend status code.
- **Liability:** The owner signs for a visibility change in four monitoring functions and for a new role OID field in PgStat_Backend that future per-backend functions must check correctly. The change is small and limited to HEAD.
- **Author follow-up:** The author revised the patch through v12 in about two months and dropped two patches at the committer's request, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 16 KiB, touches only the statistics code, docs and one regression test, and copies an existing rule. A committer who knows the area is engaged and a reviewer tested v12 without findings.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Two sources of the owning role for one family of functions. Refuted (already handled): Message #23 says v11-0001, kept as v12, adds a comment to pgstat_fetch_stat_backend_by_pid() that cached statistics may belong to an older backend. A future cumulative function calls that fetch function, and the commit message states that the check uses the role stored with the data.

### Review angles

- prior-attempts: 0 objection(s). The only listed precedent is a restriction with no real security gain, and this patch has that shape too, since pgsql-security judged it not a vulnerability. But it removes an exception to an existing rule, keeps access for owners and pg_read_all_stats, and a committer asked for it in message #3. The thread does not mention any earlier attempt at the same idea.
- standards-compat: 0 objection(s). From the standards and compatibility angle the change follows existing practice: the four functions adopt the HAS_PGSTAT_PERMISSIONS() rule and the NULL or no-rows convention that their sibling functions and pg_stat_activity already use, it adds no syntax, GUC, catalog object or pg_dump state, and it is limited to HEAD. Unprivileged monitoring roles that read per-backend IO, WAL or lock numbers of other sessions will now get empty results without an error. That needs a release note entry, but it is the same behaviour pg_stat_activity already has. [P9] is about restricting EXPLAIN without a security gain, and the consistency argument for this patch was accepted in messages #3 and #8.
- replication-ondisk: 0 objection(s). The only persistent change is a role OID field in PgStat_Backend, on HEAD only, with no WAL, catalog, protocol or pg_upgrade effect. Per-backend stats entries are dropped at backend exit, the reset callback keeps the field, and whether a stats file format bump is needed can only be checked in the code.
- security: 0 objection(s). The patch tightens access, follows the existing HAS_PGSTAT_PERMISSIONS() rule used by pg_stat_activity and the sibling functions, and checks against the role stored in PgStat_Backend, which closes the snapshot case from message #13. [P9] does not apply because per-backend IO, WAL, lock and subxact numbers are not available elsewhere, and the subxact race was dropped in message #25 as existing pg_stat_activity behaviour.
- maintenance-fit: 1 objection(s). Otherwise the change is small and copies existing practice. Michael Paquier is engaged and steered the design, and the costs fall only on callers of these functions. Nothing touches a hot path or adds a dependency.
- already-exists: 0 objection(s). No existing feature gives the same result. Revoking EXECUTE from PUBLIC would also hide a role's own sessions, while HAS_PGSTAT_PERMISSIONS() is the existing check the sibling functions already use, so the patch extends that check and adds no second mechanism.
- demand: 0 objection(s). This proposal tightens access on four existing functions to match their sibling functions and pg_stat_activity. It adds no feature, so the demand angle does not apply. pgsql-security asked for the consistency fix (message #1), and Michael Paquier and Bertrand Drouvot support it (messages #3, #8). Monitoring roles keep access through pg_read_all_stats. [P9] declined a restriction on EXPLAIN (ANALYZE) that brought no real security gain, but it does not fit here. That restriction removed a capability from all users, while this one only aligns four functions with an access rule that already covers the same data.

### Precedents retrieved

- P1: [Add tracking of backend memory allocated to pg_stat_activity](https://commitfest.postgresql.org/patch/3865/) (cf, Withdrawn, 2023-01-05) - judged not relevant; reason summarised from its thread
- P2: [Get rid of pgstat_count_backend_io_op*() functions](https://commitfest.postgresql.org/patch/6025/) (cf, Rejected, 2026-03-25) - judged not relevant; reason summarised from its thread
- P3: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant; reason summarised from its thread
- P4: [Add checkpoint stats of snapshot and mapping files of pg_logical dir](https://commitfest.postgresql.org/patch/3389/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P5: [Adjust pg_stat_get_lock() prorows to match lock types](https://commitfest.postgresql.org/patch/6740/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P6: [Eliminating "Permission denied" errors on stat() (on Windows)](https://commitfest.postgresql.org/patch/3058/) (cf, Withdrawn, 2021-12-03) - judged not relevant; reason summarised from its thread
- P7: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P8: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P9: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - used in the review; reason summarised from its thread
- P10: [Check consistency of GUC defaults between .sample.conf and pg_settings.boot_val](https://commitfest.postgresql.org/patch/3736/) (cf, Returned with feedback, 2024-01-26) - judged not relevant; reason summarised from its thread
- P11: [Fix pg_stat_statements display of normalized FETCH counts](https://commitfest.postgresql.org/patch/6756/) (cf, Withdrawn, 2026-05-12) - judged not relevant
- P12: [Show WAL stats on pg_stat_io](https://commitfest.postgresql.org/patch/4416/) (cf, Returned with feedback, 2024-01-18) - judged not relevant
- P13: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P14: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P15: [Enhance statistics reset functions to return reset timestamp](https://commitfest.postgresql.org/patch/5956/) (cf, Withdrawn, 2025-10-28) - judged not relevant
- P16: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P17: [Add system views pg_permissions and pg_ownerships](https://commitfest.postgresql.org/patch/3033/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P18: [Revert "Get rid of WALBufMappingLock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c13070a27b63d9ce4850d88a63bf889a6fde26f0) (revert, Reverted, 2025-08-22) - judged not relevant
- P19: [Move tablespace path re-creation from the makefiles to pg_regress (test for mkdir function in the backend)](https://commitfest.postgresql.org/patch/3067/) (cf, Withdrawn, 2021-04-16) - judged not relevant
- P20: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P21: [Revert "pg_stat_statements: Add coverage for entry_dealloc()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1141e29b61e9bf82b045540d41b27581fc3207ec) (revert, Reverted, 2023-12-31) - judged not relevant
- P22: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P23: [Prevent incorrect entries in pg_stat_activity.](https://commitfest.postgresql.org/patch/5105/) (cf, Returned with feedback, 2025-10-17) - judged not relevant
- P24: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 106 s; cost reported by the CLI: $1.96
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:35:16.525Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
