# Design pre-review: Check the fd number against FD_SETSIZE in PQsocketPoll

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7550](https://commitfest.postgresql.org/patch/7550/) (Needs review); authors: Roman Khapov (rkhapov)
- Thread: [Check the fd number against FD_SETSIZE in PQsocketPoll](https://www.postgresql.org/message-id/flat/985BA3B4-1281-451A-AE43-6ECCD87D75E4%40yandex-team.ru), started 2026-10-29
- Messages read: 2 of 2
- Patch prose read: v2-0001-libpq-guard-select-fallback-against-fds-FD_SETSIZ.patch, 0001-libpq-guard-select-fallback-against-fds-FD_SETSIZE.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

One objection survived, maintenance-fit-1, and it is a fixable level 2 point about where the fix belongs. The guard itself replaces an out-of-bounds FD_SET() with an EINVAL failure and nobody objected to that. A committer who knows the libpq portability code has to decide between guarding the non-Windows select() fallback and removing it. Confidence is low because the thread has only the author's two messages and names no platform that compiles the branch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. I could not check how the guard is placed relative to FD_SET(), how v2 excludes Windows, or whether other select() call sites have the same gap.

## The proposal, restated

**Problem.** On platforms without poll(2), PQsocketPoll() falls back to select(2) and calls FD_SET() on the connection socket without checking its number. If the socket fd is at or above FD_SETSIZE, FD_SET() writes past the fd_set on the stack. This affects libpq client applications that hold many open files before connecting.

**Proposed design.** In the select() fallback, PQsocketPoll() checks the socket fd against FD_SETSIZE and fails with EINVAL instead of calling FD_SET(). v2 skips the check on Windows, where fd_set is a bounded array and FD_SET() checks its limits. The change is in src/interfaces/libpq/fe-misc.c only.

**User-visible surface.**
- PQsocketPoll() can now fail with EINVAL when the socket fd is at or above FD_SETSIZE, on non-Windows platforms without poll(2).

**What the author claims.**
- The unchecked FD_SET() in the select() fallback can corrupt the stack.
- The fix is cheap and worth having as a defence.
- The issue is mostly theoretical, since modern systems have poll(2) and do not compile this branch.
- Testing on Windows showed the check is not needed there.

**Thread state.** Only the author has posted: the first message with v1 and a follow-up with v2 that drops Windows. No reviewer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Guard goes into a branch no supported Unix may compile

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

v2 adds a new error path to the non-Windows select() fallback in PQsocketPoll(). If no buildfarm animal builds that branch, a later change to fe-misc.c can break it with no one noticing. The first report then comes from a user on a rare platform, and no committer can reproduce it.

Message #2 says the remaining case is "mostly theoretical", and the proposal names no supported platform that lacks poll(2). Code that is never compiled or run in testing costs review time now and can rot later. A committer would likely ask whether the right change is to require poll(2) on non-Windows platforms and drop that branch, which removes the risk instead of guarding it. The guard is small, so this is about where the fix goes.

Thread: [#1 (Roman Khapov, 2026-10-29)](https://www.postgresql.org/message-id/985BA3B4-1281-451A-AE43-6ECCD87D75E4%40yandex-team.ru), [#2 (Roman Khapov, 2026-10-29)](https://www.postgresql.org/message-id/EA35FD13-9785-41D0-8D2F-FB3EE6A35BB0%40yandex-team.ru)

**Question:** Which supported platform or buildfarm animal compiles the non-Windows select() fallback today, and if none does, why keep and extend that branch instead of removing it?

Counter-check: held. Message #2 calls the remaining case "mostly theoretical", and the material names no supported non-Windows platform without poll(2). Windows builds compile the select() branch, but v2 skips the guard there, so nothing builds or tests the new check.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | PQsocketPoll() is a public libpq function and gains a new EINVAL failure case; its signature does not change. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch replaces an out-of-bounds stack write in the client-side select() fallback with an error return. |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix, so a committer has to decide whether to backpatch the guard. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The new error path sits in a select() branch that, per message #2, no known non-Windows platform compiles. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which supported platform or buildfarm animal compiles the non-Windows select() fallback in PQsocketPoll() today?
2. If no supported platform compiles that branch, why guard it instead of requiring poll(2) on non-Windows builds and removing it?
3. Do other select() call sites in libpq or the client tools call FD_SET() on an unchecked fd and need the same check?
4. Is the guard meant for backpatching, as the Bugfix tag suggests, or for master only?
5. How can the new EINVAL path be tested if no buildfarm animal builds the branch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows libpq's socket code and the platform portability rules (which systems must have poll(2)) would have to own it. No reviewer has replied in the thread.
- **Liability:** A new error path in a branch that may never be compiled or tested, for as long as the select() fallback stays in the tree. The size is small: one check in one file.
- **Author follow-up:** The author posted v2 the same day after testing on Windows, which suggests responsiveness. The thread is two weeks old and has no reviewer feedback to respond to yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 1.4 KB in one file, and message #2 calls the remaining case "mostly theoretical". The open point is whether to guard the non-Windows select() fallback or remove it, and that should be settled before code review.

## Appendix

### Refuted in the counter-check

- `demand-1` No supported platform named that compiles the guarded branch. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about the untested branch and also states the alternative of requiring poll(2) on non-Windows builds.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P13] concerns pgbench connection limits on Windows, a different program, and records no reason, so nothing from it applies here. Message #1 points to a separate pgbench FD_SETSIZE thread as the starting point, and the material does not report how that thread ended.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. PQsocketPoll() already returns -1 with errno set on failure, so EINVAL fits that contract and replaces undefined behaviour; no syntax, GUC, catalog, pg_dump or psql surface changes, and only builds without poll(2) are affected.
- replication-ondisk: 0 objection(s). The patch changes only the select() fallback in libpq's fe-misc.c, so there is nothing for this angle to review. It touches no on-disk format, WAL, catalog, replication or wire protocol message, and PQsocketPoll() keeps its signature and its existing -1-with-errno failure convention, so ABI and backpatching are unaffected.
- security: 0 objection(s). The patch adds no privilege, role, setting or exposure, and only replaces an out-of-bounds FD_SET() in the client-side select() fallback with an EINVAL failure. That makes the code safer, and the only fd numbers involved belong to the client process, so I see no security objection.
- maintenance-fit: 1 objection(s). The only objection concerns where the fix belongs. The change is small, touches one file, and adds no cost on any hot path. [P13] has no recorded reason and covers pgbench on Windows, so I do not cite it.
- already-exists: 0 objection(s). No existing Postgres feature, setting or extension guards PQsocketPoll() against fds at or above FD_SETSIZE in its select() fallback, so this angle has nothing to raise. Whether the branch should be removed outright, given that the author calls it theoretical, belongs to the desirability and design angles.
- demand: 1 objection(s). One objection only. The change is small and a memory-safety guard, so the cost is low. The open point is whether any supported user can reach the guarded code.

### Precedents retrieved

- P1: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P2: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P3: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant; reason summarised from its thread
- P4: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P5: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant; reason summarised from its thread
- P6: [Tab completion for SELECT in psql](https://commitfest.postgresql.org/patch/1376/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P7: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant; reason summarised from its thread
- P8: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Revert misguided change to postgres_fdw FOR UPDATE/SHARE code.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8ec8760fc87ecde0516e511f1c55aec627b01ea7) (revert, Reverted, 2014-12-12) - judged not relevant
- P10: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant; reason summarised from its thread
- P11: [Fix out-of-bounds with function PQescapeInternal](https://commitfest.postgresql.org/patch/4910/) (cf, Rejected, 2024-04-04) - judged not relevant; reason summarised from its thread
- P12: [Kerberos delegation support in libpq and postgres_fdw](https://commitfest.postgresql.org/patch/3265/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P13: [pgbench stopped supporting large number of client connections on Windows](https://commitfest.postgresql.org/patch/2827/) (cf, Returned with feedback, 2021-04-08) - used in the review
- P14: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - judged not relevant
- P15: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant
- P16: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P17: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P18: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant
- P19: [Refactoring postgres_fdw/connection.c](https://commitfest.postgresql.org/patch/3782/) (cf, Withdrawn, 2023-05-23) - judged not relevant
- P20: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant
- P21: [Respect client-initiated CopyDone during logical streaming replication](https://commitfest.postgresql.org/patch/621/) (cf, Withdrawn, 2019-07-08) - judged not relevant
- P22: [Avoid obscure DES key buffer loop bounds in pgcrypto](https://commitfest.postgresql.org/patch/6953/) (cf, Withdrawn, 2026-09-25) - judged not relevant
- P23: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant
- P24: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 80 s; cost reported by the CLI: $0.57
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:08:26.344Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
