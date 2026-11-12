# Design pre-review: Fall back to serial vacuum when out of DSM segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7380](https://commitfest.postgresql.org/patch/7380/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [Parallel autovacuum: leader crashes when no DSM segment can be created](https://www.postgresql.org/message-id/flat/CALj2ACVjBN03Fq4XWgKH1zOfkapK6-P3ufNLHUHnZcW1zPm%2BMg%40mail.gmail.com), started 2026-09-28
- Messages read: 7 of 7
- Patch prose read: v1-0001-Fix-crash-in-parallel-autovacuum-when-no-DSM-segm.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No fundamental objection survived for HEAD. The fallback copies what parallel index builds do, adds no user-visible surface, and costs nothing for users who do not run parallel vacuum. The one surviving objection, demand-1, is fixable and bears mostly on the "stable" target, which the thread does not argue. A committer who knows parallel vacuum must decide on the backpatch and on removing the nworkers check from cc053b6e127.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The follow-up patch and messages #4 to #7 were not shown. Without the code, it is not possible to confirm that removing the nworkers check is safe or that every caller handles a NULL return from parallel_vacuum_init().

## The proposal, restated

**Problem.** When all DSM slots are taken, VACUUM (PARALLEL 2) fails with "too many dynamic shared memory segments", while VACUUM (PARALLEL 0) succeeds. InitializeParallelDSM() already falls back to a leader-only context, but parallel vacuum then creates a shared TidStore that needs another DSM segment. This affects manual parallel vacuum and parallel autovacuum.

**Proposed design.** When pcxt->seg is NULL, parallel_vacuum_init() returns NULL, so the caller uses a local TidStore and vacuums serially, like parallel index builds do. The patch also removes the nworkers check added by the crash fix cc053b6e127, which the author thinks becomes redundant.

**User-visible surface.**
- VACUUM (PARALLEL n) and parallel autovacuum finish serially instead of failing when no DSM segment is free at context setup.

**What the author claims.**
- The shared TidStore in the no-worker case defeats the leader-only fallback from InitializeParallelDSM().
- Parallel index builds already back out in the same way.
- The nworkers check from cc053b6e127 becomes redundant with this change.

**Thread state.** The crash fix (v1 from message #1) was pushed as cc053b6e127; this entry is the follow-up. Bharath objects that a later dead-TID DSM allocation can still fail, so failing is fine, but supports pursuing it for HEAD; shihao zhong says v1 still fixes the all-slots-taken case and suggests reserving DSM slots for vacuum.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No field report of DSM exhaustion breaking parallel vacuum

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), backpatchability · `demand-1`

A committer pushes the fallback, possibly to stable branches as the CommitFest target suggests. No user who reported the error exists to confirm that it helps. Users who do run out of DSM slots, and still have one slot left, still see "too many dynamic shared memory segments" from the dead-TID allocation.

The only evidence is a synthetic test in message #4, where another session holds every DSM slot. Message #7 calls DSM exhaustion very rare and says the reviewer had never seen it. Message #6 says the error is acceptable because it can happen anyway, and message #7 agrees that the one-slot-left case still fails. So the fix covers a narrow case with no reported users. The change is small, but a Bugfix tag with target "stable" needs more than a synthetic case.

Thread: [#4 (Ayush Tiwari, 2026-09-30)](https://www.postgresql.org/message-id/CAJTYsWVvGOGmyXVjjOwC4-R31Lp1iay27jR5tukr%2BCTh2JefMA%40mail.gmail.com), [#6 (Bharath Rupireddy, 2026-10-23)](https://www.postgresql.org/message-id/CALj2ACUO8CFQoN9PRenxiuYHXE1hVBHd2_OqHvEUagXO-C-VdQ%40mail.gmail.com), [#7 (shihao zhong, 2026-10-30)](https://www.postgresql.org/message-id/CAGRkXqQHk9VjvXyRwTvOnJBv1K276uXCyDLb0kDvNjSHFFMeAQ%40mail.gmail.com)

**Question:** Which production systems have seen parallel vacuum or parallel autovacuum fail with "too many dynamic shared memory segments", and would the all-slots-taken fallback have covered them?

Counter-check: held. The thread shows only a synthetic case, message #7 calls DSM exhaustion very rare, and the "stable" target is not argued anywhere. For HEAD the objection is weak: the fallback copies parallel index builds, adds no user-visible surface, and costs non-users nothing.

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
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix with target "stable", but the thread discusses only HEAD. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch removes the nworkers check that crash fix cc053b6e127 added in vacuumparallel.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No field report of DSM exhaustion breaking parallel vacuum |

## 5. Questions the author must answer before anyone reviews code

1. Is this change meant for HEAD only, or for backpatch to 19 as the "stable" target on the CommitFest entry suggests?
2. Which production systems have seen parallel vacuum or parallel autovacuum fail with "too many dynamic shared memory segments", and would the all-slots-taken fallback have covered them?
3. What happens when the parallel context gets a segment but the later dead-TID DSM allocation fails: does vacuum still raise the error, and is that accepted?
4. Which case did the nworkers check from cc053b6e127 cover, and why can it not occur once parallel_vacuum_init() returns NULL for a NULL pcxt->seg?
5. Does vacuum log or report that it fell back to serial because no DSM segment was free?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Masahiko Sawada pushed the related crash fix in message #3. The follow-up needs a committer who knows parallel vacuum and the DSM fallback in InitializeParallelDSM().
- **Liability:** The owner signs for parallel vacuum and parallel autovacuum behaviour under DSM exhaustion, including the removal of the guard from cc053b6e127. A backpatch extends that to the stable branch for its lifetime.
- **Author follow-up:** The author created the CommitFest entry on 2026-10-01 and renamed it on 2026-10-21. The visible messages show no other sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The thread shows DSM exhaustion only in a synthetic test, so few users benefit. The change is local to parallel_vacuum_init() and follows the existing parallel index build fallback; the follow-up patch itself was not read.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the index returned no earlier rejected, returned or reverted proposal on this idea. Inside the thread, the only earlier work is the crash fix from message #1, pushed in message #3. That fix complements this proposal and was not rejected, and the open concern in message #6 belongs to the design angle.
- standards-compat: 0 objection(s). No standards or compatibility objection holds up. The proposal adds no syntax, GUC, catalog object or dump state, and it turns an error into the leader-only fallback that InitializeParallelDSM() already offers and that VACUUM (PARALLEL) documents, since fewer workers than requested is normal. The "stable" target and whether the fallback is reported are open questions, but neither gives a concrete failure for existing users.
- replication-ondisk: 0 objection(s). The proposal touches no persistent or cross-node state: no on-disk format, WAL, catalog, wire protocol or exported signature changes. DSM segments are transient shared memory, and choosing a local TidStore over a shared one does not affect replication, pg_upgrade or the extension ABI.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes only whether VACUUM (PARALLEL n) and parallel autovacuum fail or fall back to serial when DSM slots run out, and that fallback makes an existing unprivileged way of using up DSM slots less harmful.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds. The change is local to parallel_vacuum_init(), copies the existing leader-only fallback that parallel index builds use, and adds no cost for anyone who does not run parallel vacuum. Two questions are left to other angles: whether removing the nworkers guard from cc053b6e127 is safe, and whether the fix is incomplete because a later dead-TID DSM allocation can still fail (message #6).
- already-exists: 0 objection(s). Postgres has no existing way to meet this need. A user can rerun with VACUUM (PARALLEL 0), but parallel autovacuum has no such retry. The patch also does not add a new mechanism: it makes the existing leader-only fallback in InitializeParallelDSM() reach vacuum, as parallel index builds already do.
- demand: 1 objection(s). One demand objection. The change itself is small and costs non-users almost nothing, so this is a "not now, show demand" point for the stable target, and the case for HEAD is weaker.

### Precedents retrieved

- P1: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant; reason summarised from its thread
- P2: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/1223/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P3: [Autovacuum dynamic cost_limit, cost_delay](https://commitfest.postgresql.org/patch/2983/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P4: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/734/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [Block level parallel vacuum](https://commitfest.postgresql.org/patch/954/) (cf, Returned with feedback, 2017-03-05) - judged not relevant; reason summarised from its thread
- P7: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant; reason summarised from its thread
- P8: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P9: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P10: [Revert recent change to RequestNamedLWLockTranche().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d814d7fc3d5257ae258b502229fc7ca97c97270a) (revert, Reverted, 2025-09-04) - judged not relevant
- P11: [Initial mach based shared memory support.](https://commitfest.postgresql.org/patch/2843/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P12: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P13: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P14: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P15: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant
- P16: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P17: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P18: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant
- P20: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant
- P21: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant
- P22: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P23: [Introduce some randomness to autovacuum](https://commitfest.postgresql.org/patch/5732/) (cf, Rejected, 2026-06-07) - judged not relevant
- P24: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 112 s; cost reported by the CLI: $0.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:48:02.585Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
