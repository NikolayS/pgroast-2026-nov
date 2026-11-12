# Design pre-review: Report userid to PgBackendStatus for standalone backends

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7277](https://commitfest.postgresql.org/patch/7277/) (Needs review); authors: ChangAo Chen (chenchangao)
- Thread: [\[PATCH\] Report userid to PgBackendStatus for standalone backends](https://www.postgresql.org/message-id/flat/tencent_18F1100EB0ECCA42510B516A835214640409%40qq.com), started 2026-09-08
- Messages read: 3 of 3
- Patch prose read: v1-0001-Report-userid-to-PgBackendStatus-for-standalone-b.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Only demand-1 survived, and it fails level 1b on demand. Message #2 asked who benefits when no other process can read the status of a postgres --single backend, and message #3 answered that the empty user name looks strange but is harmless. The change is small and costs users of normal servers nothing, so this is a "not now, show demand" outcome and no rejection of the idea.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody could check whether bootstrap mode (initdb) takes the same path, or at which point in backend status setup the user ID is recorded.

## The proposal, restated

**Problem.** In single-user mode (postgres --single), pg_stat_activity shows no user name for the session. The author found this by querying pg_stat_activity in single-user mode and calls it strange but harmless. The thread names no user who needs the information.

**Proposed design.** The standalone backend always runs as the bootstrap superuser, and the patch records that user ID in its PgBackendStatus entry. Only src/backend/utils/activity/backend_status.c changes.

**User-visible surface.**
- pg_stat_activity in single-user mode shows usesysid and usename for the session instead of empty values.

**What the author claims.**
- Standalone backends always use the bootstrap superuser.
- Reporting it lets users see the user in pg_stat_activity even in single-user mode.
- The current empty user name is strange but harmless.

**Thread state.** Michael Paquier asked why this matters, since no other process can read the status of a postgres --single backend. The author replied that the empty user name only looks strange; nobody else has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No user is shown to need the user name in single-user mode

fixable · level 1b, rubric item 2 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A DBA runs postgres --single to recover from wraparound and queries pg_stat_activity. Only that one session can see its own row, and the DBA already knows it runs as the bootstrap superuser. The patch gives this user nothing new, and a committer still takes on a change to backend status startup.

Message #2 asked why this matters when no other process can read the status of a postgres --single backend. Message #3 does not name a use. It says the empty user name "looks a bit strange though harmless" and that the author noticed it while querying in single-user mode. The thread contains no field report, no request from another user, and no tool that reads usename in single-user mode. The patch is small, but small cost does not replace a stated user problem.

Thread: [#2 (Michael Paquier, 2026-09-08)](https://www.postgresql.org/message-id/aqCaBrpR7jsa8yxo%40paquier.xyz), [#3 (cca5507, 2026-09-09)](https://www.postgresql.org/message-id/tencent_787867078376F4A11C8558B57829CF327308%40qq.com)

**Question:** Which tool or recovery procedure reads usesysid or usename from pg_stat_activity in single-user mode, and what goes wrong for it today?

Counter-check: held. Message #2 asked who benefits, and message #3 answered only that the empty user name "looks a bit strange though harmless". The change is small and costs non-users nothing, but the thread names no user, tool or procedure that needs it.

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
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a case for standalone backends to backend status setup in backend_status.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Only the single-user session can read its own pg_stat_activity row, and the thread names no user who needs usename there. |

## 5. Questions the author must answer before anyone reviews code

1. Which tool or recovery procedure reads usesysid or usename from pg_stat_activity in single-user mode, and what goes wrong for it today?
2. Does bootstrap mode (initdb) take the same code path, and what does the patch record there?
3. Are other pg_stat_activity columns also empty or wrong in single-user mode, and why does the patch fill only the user ID?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier replied in message #2 and questioned the need; the owner would be someone who knows the backend status and activity reporting code.
- **Liability:** One extra case for standalone backends in backend status setup, kept correct if that setup or bootstrap mode changes later. The liability is small and has no on-disk or wire consequence.
- **Author follow-up:** The author answered the committer's question within a day in message #3; no sign either way about post-commit fixes.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A named tool, monitoring query or recovery procedure that reads usesysid or usename from pg_stat_activity in single-user mode and misbehaves on the empty value, or a report from another user who hit this.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch changes one file and is 1.1 KiB, so cost is low, but it affects only postgres --single sessions and message #3 calls the current state harmless. It is worth reviewer time once a use is shown.

## Appendix

### Refuted in the counter-check

- `already-exists-1` session_user already answers this inside the single-user session. Refuted (not specific): Duplicate of demand-1. Both rest on message #2 and the "harmless" reply in message #3; the session_user point answers a need the author did not claim, since the proposal is about an empty column looking strange.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus returned no earlier attempt at this idea, and no thread message points to an earlier discussion. Desirability, raised by Michael Paquier in message #2, belongs to another angle.
- standards-compat: 0 objection(s). The patch adds no syntax, GUC, catalog entry or dumped object, so pg_dump and psql are unaffected. The only visible change fills usesysid and usename in pg_stat_activity during single-user mode, and no plausible script depends on those columns staying empty. Not checked without the code: whether bootstrap mode (initdb) takes the same path.
- replication-ondisk: 0 objection(s). The patch changes only in-memory backend status (PgBackendStatus in shared memory) for a standalone backend. It touches no on-disk format, WAL, catalog, wire protocol or replication state, and no other process reads this entry, so this angle has nothing to object to.
- security: 0 objection(s). No privilege or exposure aspect. A single-user backend runs as the bootstrap superuser with no other sessions, so recording that user ID in its own status entry shows nothing to anyone who could not already see it.
- maintenance-fit: 0 objection(s). From the maintenance angle there is nothing worth sending. The patch changes one file and only fills a field once for a standalone backend, so users of normal multi-user servers pay no hot-path cost and later features need no new touch point. The weak case for the change belongs to the desirability angle, which Michael Paquier raised in message #2. Without the code I could not check whether bootstrap mode (initdb) takes the same path.
- already-exists: 1 objection(s). One objection only. The change is small and its cost is low, so the objection is about desirability and is not fatal.
- demand: 1 objection(s). One demand objection only. The cost looks small, so the honest outcome is "not now, show demand" rather than rejection.

### Precedents retrieved

- P1: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P2: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P3: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P5: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P6: [Display individual query in pg_stat_activity](https://commitfest.postgresql.org/patch/2656/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P7: [Prevent incorrect entries in pg_stat_activity.](https://commitfest.postgresql.org/patch/5105/) (cf, Returned with feedback, 2025-10-17) - judged not relevant; reason summarised from its thread
- P8: [Addition of authenticated ID to pg_stat_activity](https://commitfest.postgresql.org/patch/3095/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant; reason summarised from its thread
- P10: [pg_stat_activity: make slow/hanging authentication more visible](https://commitfest.postgresql.org/patch/4974/) (cf, Withdrawn, 2025-03-21) - judged not relevant
- P11: [New Table Access Methods for Multi and Single Inserts](https://commitfest.postgresql.org/patch/2871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P13: [Add %r substitution for psql prompts to show recovery status](https://commitfest.postgresql.org/patch/1402/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P14: [Move bki file pre-processing from initdb to bootstrap](https://commitfest.postgresql.org/patch/4543/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [REINDEX backend filtering](https://commitfest.postgresql.org/patch/2895/) (cf, Withdrawn, 2021-05-08) - judged not relevant
- P16: [Implementing OCSP Stapling for Server certificate status verification](https://commitfest.postgresql.org/patch/4849/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P17: [Add document is_superuser](https://commitfest.postgresql.org/patch/3889/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P18: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P19: [Have BackendXidGetPid return pid_t](https://commitfest.postgresql.org/patch/6229/) (cf, Rejected, 2026-01-07) - judged not relevant
- P20: [Fix incorrect handling of umid in postgres_fdw](https://commitfest.postgresql.org/patch/622/) (cf, Returned with feedback, 2016-08-19) - judged not relevant
- P21: [REINDEX filtering in the backend](https://commitfest.postgresql.org/patch/2206/) (cf, Returned with feedback, 2019-09-06) - judged not relevant
- P22: [Orphaned users in PG16 and above can only be managed by Superusers](https://commitfest.postgresql.org/patch/5608/) (cf, Withdrawn, 2026-03-05) - judged not relevant
- P23: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 102 s; cost reported by the CLI: $0.67
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 50 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:36:08.754Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
