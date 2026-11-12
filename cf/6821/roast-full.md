# Design pre-review: Unify parallel worker handling for index builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6821](https://commitfest.postgresql.org/patch/6821/) (Needs review); authors: Lukas Fittl (lfittl)
- Thread: [Unify parallel worker handling for index builds and instrumentation](https://www.postgresql.org/message-id/flat/CAP53Pkw3WcVbg61G29AmN%3D6XkuxGryoWKnhTyC1JmkT08i8qVA%40mail.gmail.com), started 2026-05-31
- Messages read: 7 of 7
- Patch prose read: v2-0001-Fix-query-ID-reporting-in-parallel-workers-for-GI.patch, v2-0005-Unify-parallel-index-build-scan-handling.patch, v2-0002-Fix-broken-HOT-chain-handling-in-parallel-BRIN-an.patch, v2-0004-Unify-parallel-worker-handling-for-query-text.patch, v2-0003-Unify-parallel-index-build-infrastructure.patch, v1-0001-instrumentation-Use-Instrumentation-struct-for-pa.patch, v1-0003-Unify-parallel-worker-handling-for-query-text.patch, v1-0002-Unify-parallel-worker-handling-for-instrumentatio.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low)

No objection survived the critic. The only objection raised, maintenance-fit-1, was refuted because Postgres does not keep its C API stable across major versions. The series fixes two reported bugs and removes duplicated code, with support in messages #2 and #3. Confidence is low because nobody has reviewed v2, and the wrong-results claim has no reproducing test in the material.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the wrong-results claim, the safety of the 0002 backpatch for extension ABI, and the behaviour of ParallelIndexBuildScan() in 0005 are unchecked. Nobody has reviewed v2 yet.

## The proposal, restated

**Problem.** nbtree, GIN and BRIN each carry near-identical code for parallel worker setup, shared memory and leader-worker communication, and parallel vacuum duplicates the query text handling. This duplication has hidden bugs: GIN workers report query ID 0, and parallel BRIN and GIN builds never set indcheckxmin after a broken HOT chain, so older snapshots can use the index and get wrong results.

**Proposed design.** Two bug fixes come first: GIN workers report the leader's query ID, and BRIN and GIN pass the broken HOT chain flag back to the leader, backpatched to 17 and 18. Then a new parallel_index_build module gives AMs shared helpers for context creation, shared memory, worker wait, abort and end, and a common ParallelIndexBuildScan() records broken HOT chains itself. Separate helpers in parallel.c handle query text for index builds and vacuum.

**User-visible surface.**
- New C API header access/parallel_index_build.h with ParallelIndexBuildCreateContext (takes library_name), ParallelIndexBuildReportScanDone, ParallelIndexBuildWaitForWorkers, ParallelIndexBuildAbort, ParallelIndexBuildEnd.
- New C API ParallelIndexBuildScan() that joins the shared scan, runs table_index_build_scan() and records broken HOT chains.
- New query text helpers in access/parallel.h, including RestoreParallelQueryText(), for parallel workers.
- pg_stat_activity shows the leader's query_id for parallel GIN build workers instead of 0.
- Parallel BRIN and GIN builds now set indcheckxmin when they meet a broken HOT chain.

**What the author claims.**
- Parallel BRIN and GIN builds fail to set indcheckxmin, so a transaction with an older snapshot can get wrong results.
- The HOT chain fix should be backpatched to 17 for BRIN and 18 for GIN.
- ParallelIndexBuildEnd now requires workers to report WAL and buffer usage, which the author finds acceptable for extensions.
- A sketched pgvector port shows a net code reduction, plus a shim of about 330 lines for older versions.
- pgvector has the same query ID and broken HOT chain bugs, and the new helpers fix them.

**Thread state.** Sami Imseih and Michael Paquier supported the query text and index build helpers; Michael objected to the larger Instrumentation struct in shared memory, so v1 0001 and 0002 were dropped. v2 adds two bug fixes and ParallelIndexBuildScan(), and nobody has reviewed v2 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Patch 0002 makes parallel BRIN and GIN builds set the existing pg_index.indcheckxmin flag; no catalog definition changes. |
| ABI / extension API | **yes** | New header access/parallel_index_build.h and query text helpers in access/parallel.h are usable by out-of-core index AMs. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | With indcheckxmin set, transactions with older snapshots no longer use a newly built BRIN or GIN index. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Patch 0002 proposes a backpatch to 17 for BRIN and 18 for GIN. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new shared module in access/common becomes the path for parallel builds in nbtree, GIN and BRIN. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is there a test or script that reproduces wrong results from a parallel BRIN or GIN build that met a broken HOT chain?
2. Which patches go to the back branches, and do the 17 and 18 versions of 0002 differ from the master version?
3. Is the GIN query ID fix in 0001 also meant for backpatch to 18?
4. How does an out-of-core AM learn that ParallelIndexBuildEnd now requires workers to report WAL and buffer usage?
5. Can an AM keep its own fields under the shared mutex, or must it add a second lock as the pgvector sketch does?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier supported the query text and index build helpers in message #3, but only for v1. The backpatched fix in 0002 needs someone who knows index builds, HOT chains and indcheckxmin.
- **Liability:** The owner signs for a correctness fix on stable branches 17 and 18, and for a shared module that every parallel index build in nbtree, GIN and BRIN runs through from now on.
- **Author follow-up:** The author posted v2 after review, dropped the two patches Michael Paquier objected to, and moved the CommitFest entry forward on 2026-10-31.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Patch 0002 fixes a reported wrong-results case in parallel BRIN and GIN builds, and the refactoring removes duplicated code without new user-facing surface. Review cost is medium because 0003 and 0005 are large and touch three index AMs; 0001 and 0002 are small and can go first.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` New extension-facing build API may need to change again soon. Refuted (no failure scenario): Postgres does not keep its C API stable across major versions, and extensions that opt in to these helpers already carry per-version shims, so a later API change is routine cost. Message #7 shows the helpers fit three in-core AMs and a pgvector port.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved list has no earlier attempt at shared parallel index build helpers or at the GIN and BRIN fixes. The thread also points to no earlier discussion of this idea, so no prior rejection reason can be applied.
- standards-compat: 0 objection(s). The series adds no SQL syntax, GUC, catalog column, psql feature or dump-relevant object, so the user-facing contract barely moves. The visible changes are bug fixes: GIN workers now show the leader's query_id, and BRIN and GIN builds now set indcheckxmin as nbtree already does. After a minor upgrade, some older-snapshot transactions may skip a newly built index, which is the documented and correct behaviour. Extension API questions belong to another angle.
- replication-ondisk: 0 objection(s). The series adds no on-disk, WAL, wire protocol or catalog change. The 0002 fix only sets the existing pg_index.indcheckxmin flag, and that flag reaches standbys through normal catalog WAL. Leader-to-worker state lives in per-build DSM between processes of one server binary, so mixed versions cannot meet. The structs 0002 changes for 17 and 18 appear to be private to brin.c and gininsert.c, so the backpatch should not break extension ABI; this needs checking against the code. Indexes built in parallel before the fix need no repair, because indcheckxmin only protects snapshots older than the build. The new C API exists only on master. The nbtree VACUUM invariant does not apply, because the patch changes index build code, not VACUUM.
- security: 0 objection(s). The proposal adds no new privilege or exposure surface: query text and query_id reach pg_stat_activity under existing visibility rules, and library_name follows CreateParallelContext. The indcheckxmin fix concerns correctness of results, not access control, and no new memory use falls under user control.
- maintenance-fit: 1 objection(s). The nbtree VACUUM invariant does not apply because this proposal changes index builds only. Committer Michael Paquier supported the index build and query text helpers in message #3, but nobody has reviewed v2 or 0005 yet.
- already-exists: 0 objection(s). Postgres has no shared parallel index build helper today. The patches build on CreateParallelContext, shm_toc and table_index_build_scan() and do not replace them, so they add no second mechanism. The BRIN and GIN fixes cover bugs that no current feature or extension handles.
- demand: 0 objection(s). Demand holds up. The bug fixes address wrong query results and a wrong query_id, and the refactoring has support from a committer (message #3) and another reviewer (message #2). It also has a sketched port by one out-of-core index AM, pgvector (message #7), which has the same bugs. Whether the new extension API should be public is a design question for other angles.

### Precedents retrieved

- P1: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P2: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant; reason summarised from its thread
- P3: [Revert "Fix parallel-safety check of expressions and predicate for index builds"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=099ca50bd41c1a72fe5a086bb095858ea74c88de) (revert, Reverted, 2024-03-06) - judged not relevant
- P4: [improve DEBUG1 logging of parallel workers for CREATE INDEX](https://commitfest.postgresql.org/patch/5490/) (cf, Withdrawn, 2025-02-03) - judged not relevant; reason summarised from its thread
- P5: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant; reason summarised from its thread
- P6: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant; reason summarised from its thread
- P7: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P9: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P10: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P11: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P12: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - judged not relevant
- P13: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P14: [Revert "Stabilize test of BRIN parallel create"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a89cd7bfcdbb45a2c23b4b6004f4503f04f83bd7) (revert, Reverted, 2024-04-17) - judged not relevant
- P15: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P16: [Fix the statistics gathering in case of commit and chain](https://commitfest.postgresql.org/patch/3657/) (cf, Returned with feedback, 2022-10-06) - judged not relevant
- P17: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [BRIN Sort - sorting using BRIN indexes](https://commitfest.postgresql.org/patch/3949/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P19: [Revert bogus fixes of HOT-freezing bug](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c6764eb3aea63f3f95582bd660785e2b0d4439f9) (revert, Reverted, 2017-11-02) - judged not relevant
- P20: [Revert current_query() change to use debug_query_string again; add comment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c175cd76a9db7d6114139a28f838697e88306ba) (revert, Reverted, 2009-01-08) - judged not relevant
- P21: [Refactoring speculative insertion with unique indexes a little](https://commitfest.postgresql.org/patch/403/) (cf, Rejected, 2016-09-20) - judged not relevant
- P22: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P23: [Turning off HOT for larger SQL queries](https://commitfest.postgresql.org/patch/30/) (cf, Returned with feedback, 2015-09-01) - judged not relevant
- P24: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0004-Unify-parallel-index-build-infrastructure.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 122 s; cost reported by the CLI: $1.71
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:04:38.645Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
