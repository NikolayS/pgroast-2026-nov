# Design pre-review: GIN index fast list search may become un-interruptible for long time

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7538](https://commitfest.postgresql.org/patch/7538/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [GIN index fast list search may become un-interruptible for long time.](https://www.postgresql.org/message-id/flat/CALdSSPisYC-zHiWJU36AP-zrPN-im-gEZrK-QD8TPn3Vb8xMvw%40mail.gmail.com), started 2026-07-04
- Messages read: 4 of 4
- Patch prose read: v1-0001-Make-GIN-scan-more-cancellable.patch, v1-0002-Move-CFI-check-before-break-condition.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated with a reproducer, and two independent production reports show demand. The surviving objections, maintenance-fit-1 and maintenance-fit-2, are fixable design points: v2 adds no interrupt check for a long pending list with few query entries, and the 64-entry cutoff has no measurements. Neither would stop a committer from reading on. v2-0002 changes when the consistent functions run relative to the lock, which needs a reviewer who knows GIN.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 patches and message #4 were not shown, so this rests on the brief's summary of v2. The locking in v2-0002 and the mapping of sorted entries to consistent-function arguments could not be checked without the code.

## The proposal, restated

**Problem.** A query that searches a GIN index with a very large key array can ignore pg_cancel_backend and statement_timeout for minutes. The backend holds a pending list buffer lock while it looks up each key item one by one, and interrupts are held off during that time. Kirill Reshke and shihao zhong report this in production.

**Proposed design.** v1 releases the pending list buffer lock and calls CHECK_FOR_INTERRUPTS() when an interrupt is pending, and moves a CFI in startScanKey() before a break. v2, by another author, sorts the query entries once and binary-searches each pending list row's keys in them, and calls the consistent functions after the lock is released. Queries with fewer than 64 entries keep the old path.

**What the author claims.**
- In Kirill's case, a 1s statement_timeout fires after 202s on master and after 1s with v2-0001.
- With a default size pending list, queries with 1000 entries run 22x faster with v2-0001.
- v2-0001 can be committed without v2-0002, and the locking is unchanged.
- Large arrays are still slow in startScanKey(), but that code holds no lock.
- Kirill wants a back-patchable fix and calls v1 uncommittable for master because of possible performance cost.

**Thread state.** Heikki called the v1 locking broken, because CHECK_FOR_INTERRUPTS() may return and the scan then continues without the lock, and he suggested sorting the keys. shihao posted v2 along those lines, and nobody has replied to v2 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Speedup leaves the uncancellable loop in place for other workloads

fixable · level 2, rubric item 9 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A site raises gin_pending_list_limit so the pending list holds millions of rows, then runs a query with 50 entries. The query takes the old path, holds the pending list buffer lock with interrupts held off, and again ignores pg_cancel_backend and statement_timeout. The DBA finds out during an incident, after the bug was believed fixed.

The bug report in message #1 is that the scan cannot be interrupted while it holds the buffer lock. v2-0001 in message #4 makes one input shape faster but adds no interrupt check, and queries below 64 entries keep the old path. The time spent still grows with pending list length, so the fix depends on the workload, and a later regression in the fast path would bring the bug back without any test failing.

Thread: [#1 (Kirill Reshke, 2026-07-04)](https://www.postgresql.org/message-id/CALdSSPisYC-zHiWJU36AP-zrPN-im-gEZrK-QD8TPn3Vb8xMvw%40mail.gmail.com), [#2 (Heikki Linnakangas, 2026-07-05)](https://www.postgresql.org/message-id/f7eeaba4-fa1d-4cfd-ab73-44ba393937b9%40iki.fi), [#4 (shihao zhong, 2026-09-25)](https://www.postgresql.org/message-id/CAGRkXqTAf2ghC9hyc_NLx5oA0PP%3Dv3L8S8LMQRcDxbr9GJ6niQ%40mail.gmail.com)

**Question:** How does v2 keep a scan cancellable when the pending list is very long but the query has few entries, or when the fast path is not used?

Counter-check: held. Message #4 says the locking is unchanged and mentions no new interrupt check, so the original complaint is answered only for large scan keys. The 50-entry scenario may take seconds, not minutes, but the material gives no measurement to rule it out.

### 2. Two lookup paths behind an unexplained 64-entry cutoff

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch changes how pending list rows are matched, for example for partial match or a new strategy, and updates only the linear path. Most regression queries use few entries, so the sorted path is rarely run. Users with large arrays get wrong or missing rows, and the bug shows up only in the field.

Message #4 keeps the old per-key lookup below 64 entries and adds a sorted binary-search path above it. Both paths must return the same matches for every opclass, and nothing fails loudly if they drift apart. The material does not say how 64 was chosen or what small queries cost on the sorted path, so a committer cannot tell whether one path would be enough.

Thread: [#4 (shihao zhong, 2026-09-25)](https://www.postgresql.org/message-id/CAGRkXqTAf2ghC9hyc_NLx5oA0PP%3Dv3L8S8LMQRcDxbr9GJ6niQ%40mail.gmail.com)

**Question:** What measurements justify keeping two lookup paths, and why would the sorted path alone not be acceptable for queries with few entries?

Counter-check: held. Size cutoffs with two code paths exist elsewhere in Postgres, and v2-0003 adds tests for the sorted path, which weakens the drift scenario. But message #4 gives no reason for 64 and no numbers for small queries, so the request for measurements stands.

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
| security / privileges | **yes** | Any user with SELECT on the table can start a scan that ignores cancel and statement_timeout; the patch narrows that. |
| backpatchability | **yes** | The entry is tagged Bugfix, Kirill wants a back-patchable fix, and Heikki says the sorting approach is not backpatchable. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2 keeps the old lookup below 64 entries and adds a sorted binary-search path above it in ginget.c. |
| performance on the hot path | **yes** | The patches change the pending list scan in ginget.c, which every GIN scan of an index with pending entries runs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does v2 keep a scan cancellable when the pending list is very long but the query has fewer than 64 entries?
2. What measurements led to the cutoff of 64 entries, and how do queries with few entries perform on the sorted path?
3. How does v2-0002 keep results correct when the consistent functions run after the pending list lock is released?
4. Is v2 meant for master only, and what change, if any, is proposed for back branches?
5. Is the v1-0002 change that moves the CFI in startScanKey() still part of the plan?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas reviewed v1 in message #2 and suggested the sorting approach; he has not yet replied to v2. The owner needs to know GIN pending list scans and buffer locking.
- **Liability:** Correct results from pending list scans for every GIN opclass, on both the linear and the sorted path, for as long as both exist. If v2-0002 goes in, also the correctness of running consistent functions after the lock is released.
- **Author follow-up:** shihao zhong offered in message #3 to review, test or write v2, then posted v2 and created the CommitFest entry. No sign yet about fixes after commit.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Two production reports describe backends that ignore cancel and statement_timeout for minutes, and the change stays inside ginget.c with no user-visible surface. Review cost is medium because the sorted path and the lock change in v2-0002 need a careful read by someone who knows GIN.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-3` No fix is proposed for released branches. Refuted (no failure scenario): The scenario is what back branches do today, and v2 does not make it worse. Fixing master first when the change is too large to back-patch is normal practice, so this is an open scope question for the thread.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P8] is about BRIN array scan keys and rejected a new opclass preprocess support procedure; v2 adds no support procedure and changes only the GIN pending list scan, so that reason does not apply. The thread cites no earlier discussion of this idea; message #1 cites only the commit that added the CFI in startScanKey().
- standards-compat: 0 objection(s). From the standards and compatibility angle, nothing in the proposal is user-facing. It adds no syntax, GUC, catalog object, dump content, psql behaviour or protocol change, and query results should stay the same. Two points sit with other angles: which fix the back branches get, and whether v2 keeps the mapping between query entries and opclass consistent-function arguments after sorting. These are backpatch planning and implementation questions, and neither gives a compatibility failure scenario.
- replication-ondisk: 0 objection(s). From the replication and on-disk angle there is nothing to object to. v1 and v2 change only how a backend scans the GIN pending list in ginget.c. Neither version changes the index page format, the pending list layout, WAL records, the catalog, the wire protocol or a public extension API, so pg_upgrade, standbys and subscribers are not affected. The buffer locking question in v2-0002 is a concurrency question for the design reviewers, not a question of persistent state.
- security: 0 objection(s). The patches add no privilege, role, GUC or user-visible surface, and no data becomes newly visible to any user. The behaviour they fix is a denial-of-service path that any user with SELECT on the table can trigger today, so the proposal shrinks the attack surface. Two questions are left for other angles: v2 does not say whether back branches get a fix, and correctness after v2-0002 releases the lock still needs to be shown.
- maintenance-fit: 3 objection(s). The fix stays inside ginget.c with no API, catalog or GUC change, and Heikki, who knows the area, is engaged. Concerns are limited to scope and two code paths.
- already-exists: 0 objection(s). Existing settings only limit exposure to the bug: fastupdate = off, a smaller gin_pending_list_limit, or calling gin_clean_pending_list(). None of them makes a running pending list scan respond to pg_cancel_backend or statement_timeout, and no extension can fix locking inside ginget.c, so a committer would not send these as a reason to reject a fix for an uncancellable backend. [P8] covers BRIN array preprocessing and its support-procedure design, so it does not apply here.
- demand: 0 objection(s). Demand is shown. Two independent reporters (message #1 and message #3) hit uncancellable GIN queries in production, and message #3 says it happened several times this year; the bug ignores pg_cancel_backend and statement_timeout, and the cost falls only on large-key GIN scans with a pending list.

### Precedents retrieved

- P1: [use AV work items infrastructure for GIN pending list's cleanup](https://commitfest.postgresql.org/patch/3060/) (cf, Returned with feedback, 2021-09-08) - judged not relevant; reason summarised from its thread
- P2: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant; reason summarised from its thread
- P3: [Revert the GinMaxItemSize calculation so that we fit 3 tuples per page.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=930fd68455576dada48521a6cef642caddea3c63) (revert, Reverted, 2014-12-30) - judged not relevant
- P4: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P5: [Revert FETCH/MOVE int64 patch. Was using incorrect checks for fetch/move in scan.l.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0e20c485617c3bcde8d9f0759d5ee1ef1c442004) (revert, Reverted, 2006-09-03) - judged not relevant
- P6: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant; reason summarised from its thread
- P7: [Fix error message for statement_timeout and fix flaky test](https://commitfest.postgresql.org/patch/5072/) (cf, Withdrawn, 2024-06-25) - judged not relevant; reason summarised from its thread
- P8: [BRIN - SK_SEARCHARRAY and scan key preprocessing](https://commitfest.postgresql.org/patch/4187/) (cf, Returned with feedback, 2024-01-15) - used in the review; reason summarised from its thread
- P9: [BufferAlloc: don't take two simultaneous locks](https://commitfest.postgresql.org/patch/3483/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P10: [Revert patch pending more discussion:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=782df58a1cafbef5defb329a7583dfa0154dfe72) (revert, Reverted, 2006-04-30) - judged not relevant
- P11: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P12: [Allow pg_verify_checksums to specify a database to scan](https://commitfest.postgresql.org/patch/1782/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P13: [Revert "Introduce CompactAttribute array in TupleDesc"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4171c44c9b791da3c00386dc6d8e6b1842e3036b) (revert, Reverted, 2024-12-03) - judged not relevant
- P14: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P15: [Fix for PL/Python slow input arrays traversal issue](https://commitfest.postgresql.org/patch/687/) (cf, Rejected, 2016-09-22) - judged not relevant
- P16: [Faster pglz compression](https://commitfest.postgresql.org/patch/2897/) (cf, Withdrawn, 2023-02-28) - judged not relevant
- P17: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant
- P18: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant
- P19: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P20: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P21: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant
- P22: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P23: [Revert copyright script changes to binary *.key files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=344c26915136fa54c4a1bf54074ba15090b32721) (revert, Reverted, 2020-01-14) - judged not relevant
- P24: [Parallel INSERT SELECT take 2](https://commitfest.postgresql.org/patch/3143/) (cf, Returned with feedback, 2022-07-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 139 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 58 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:06.709Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
