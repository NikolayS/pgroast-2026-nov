# Design pre-review: Fix SSI conflicts between index-only scans and heap writes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7344](https://commitfest.postgresql.org/patch/7344/) (Needs review); authors: Rui Zhao (zhaorui)
- Thread: [SSI can miss conflicts between index-only scans and heap writes](https://www.postgresql.org/message-id/flat/CAHWVJhFfP6YmMmQwuUb%3Da5w%2BNcSOv1Aw%2BZN5Z%2BN_yUGeyG7kHw%40mail.gmail.com), started 2026-09-24
- Messages read: 3 of 3
- Patch prose read: v3-0001-Fix-SSI-conflicts-between-index-only-scans-and-he.patch, v1-0001-Fix-SSI-conflicts-between-index-only-scans-and-he.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The problem is stated with a reproducer and is a correctness bug in SERIALIZABLE, so levels 1a and 1b pass. The one surviving objection, maintenance-fit-1, is a fixable design point: the proposal does not state the new ordering rule or say which other paths that clear the all-visible bit must follow it. Someone who knows SSI and the visibility map has to confirm that the two rechecks close the race. No one but the author has posted.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm that the two rechecks close both windows, that the writer-side check is safe where it runs, or what the visibilitymap.c change does.

## The proposal, restated

**Problem.** Two SERIALIZABLE transactions can both commit with reads that match no serial order. An index-only scan can skip the heap buffer lock while the visibility-map bit is still set, so heap_delete()/heap_update() can miss its predicate lock. A writer can also clear the bit between the reader's VM check and PredicateLockPage().

**Proposed design.** After heap_delete()/heap_update() clear an all-visible bit, they run a second conflict-in check outside the critical section and before releasing the heap buffer lock. A SERIALIZABLE index-only scan in heapam_index_getnext_slot() rechecks the VM bit after PredicateLockPage() and fetches the heap if the bit was cleared. The author says both changes are needed.

**User-visible surface.**
- Under SERIALIZABLE, some schedules that now commit will abort one transaction with SQLSTATE 40001.
- New injection-point isolation test index_only_scan_ssi in src/test/modules/injection_points.

**What the author claims.**
- Reproduced on master at 374522aa63a with GDB, for both writer-side and reader-side windows, with DELETE and UPDATE.
- With the fix, all four test cases abort one transaction with 40001.
- READ COMMITTED and REPEATABLE READ behaviour under the same schedules does not change.
- make check, the core isolation suite and the injection_points tests pass.

**Thread state.** Only the author has posted, sending v2 and v3 for test fixes; no one else has replied. CFbot flagged the patch as needing rebase on 2026-10-06, after v3.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Correctness rests on a writer-side ordering rule that nothing enforces

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch adds or reworks a heap path that clears an all-visible bit, for example a new update fast path, and omits the second conflict-in check. Every existing test still passes, and SERIALIZABLE users again commit anomalous schedules. Nobody notices until another GDB-driven report arrives.

The fix adds a second conflict check at two call sites, heap_delete() and heap_update(), and pairs it with a reader-side recheck. The commit message does not state the new invariant: any code that clears an all-visible bit while a predicate lock could be taken must recheck afterwards. The proposal also does not say whether other VM-clearing paths, or table AMs that support index-only scans, must follow the same rule. Nothing turns this rule into an assertion or a shared helper.

Thread: [#1 (Rui Zhao, 2026-09-24)](https://www.postgresql.org/message-id/CAHWVJhFfP6YmMmQwuUb%3Da5w%2BNcSOv1Aw%2BZN5Z%2BN_yUGeyG7kHw%40mail.gmail.com)

**Question:** Can the invariant be stated in a comment and put into one helper or assertion that every VM-clearing path must use, with the other paths checked against it?

Counter-check: held. Per-call-site conflict checks are existing practice in heapam.c, so the request for a shared helper or assertion is weak. The proposal still does not state the new ordering rule or say whether other paths that clear the all-visible bit, or other table AMs, need it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch touches visibilitymap.c without saying what changes; other table AMs with index-only scans may need the same rule. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | This is a bug fix for SERIALIZABLE, but the proposal does not say which supported branches have the bug. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every heap path that clears an all-visible bit must now repeat the conflict-in check, and nothing enforces that. |
| performance on the hot path | **yes** | heap_delete(), heap_update() and SERIALIZABLE index-only scans get an extra check; no measurement is given. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which other paths that clear an all-visible bit, such as heap_insert or heap_lock_tuple, have the same race, and why are they safe if left unchanged?
2. Can the commit message and a code comment state the new ordering rule between clearing the all-visible bit and the conflict-in check?
3. Which supported branches have the bug, and is the patch meant for back-patching?
4. What changes in visibilitymap.c, and does it alter any exported function signature?
5. What does the second conflict-in check and the VM recheck cost on an update-heavy workload and on a SERIALIZABLE index-only-scan workload?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the SSI predicate-lock code and the heap visibility-map locking rules would have to own this.
- **Liability:** The owner signs for the correctness of SERIALIZABLE with index-only scans, on every branch that receives the fix, and for any new serialization failures or lock-ordering problems it causes.
- **Author follow-up:** The author posted v2 and v3 within nine days to fix test failures; no rebase is visible since CFbot flagged the patch on 2026-10-06.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug affects users who rely on SERIALIZABLE, and the patch is small, about 19 KiB with tests. It touches heapam.c and SSI ordering, which few reviewers can sign off, and no one has replied in the thread.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P2] changes the SIREAD lock granularity for index-only scans and stalled on a rebase, and [P1] fixes a VACUUM and visibility-map race in GiST index-only scans with no recorded reason; neither recorded reason applies to this SSI race fix, and the thread cites no earlier discussion.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to object to. The only user-visible change is more 40001 aborts under SERIALIZABLE, which the standard requires and which SERIALIZABLE applications already have to retry. The patch adds no syntax, GUC, catalog object, pg_dump state or psql change. A committer would likely still ask which supported branches have the bug and whether the fix is meant for back-patching, but that is a question for the author, not an objection.
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, catalog, or wire protocol message: it adds an in-memory predicate-lock recheck and a visibility map bit recheck on existing paths. SERIALIZABLE does not run on hot standby, so the standby and the subscriber see no change, and the visibilitymap.c change seems to read the existing VM bit only. Without the code, I cannot check whether that change alters an exported function signature, which would matter for a back-patch.
- security: 0 objection(s). The patch adds no privilege, role, function, view or setting, and it exposes no new data or error detail; it only adds serialization failures where SSI already should raise them. Its effect on the cost and back-patching of serializable index-only scans belongs to other angles.
- maintenance-fit: 1 objection(s). The proposal fixes a real SERIALIZABLE anomaly, so desirability is not in doubt. The extra writer check runs only after an all-visible bit is cleared, and the reader recheck runs only for SERIALIZABLE index-only scans, so hot-path cost looks limited, but no measurement is given. Committability is weak because no committer who knows SSI has replied, which does not count against the idea.
- already-exists: 0 objection(s). This is a correctness fix for SSI inside heapam, and no existing feature, setting or extension gives serializable guarantees for index-only scans. Setting enable_indexonlyscan = off only avoids the bug, and only for users who know about it, so it is not a fix.
- demand: 0 objection(s). This is a correctness fix for SERIALIZABLE, where both transactions commit with reads that match no serial order, and it comes with a reproducer. A demand objection does not apply: every user who relies on SERIALIZABLE needs the guarantee, and no workaround keeps index-only scans. Cost to other workloads and back-patching belong to other angles.

### Precedents retrieved

- P1: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - used in the review; reason summarised from its thread
- P2: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - used in the review; reason summarised from its thread
- P3: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P4: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P5: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P6: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant; reason summarised from its thread
- P7: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant; reason summarised from its thread
- P8: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P9: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P10: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1419/) (cf, Returned with feedback, 2018-03-06) - judged not relevant
- P11: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P12: [Skip all-visible pages during second HeapScan of CIC](https://commitfest.postgresql.org/patch/1057/) (cf, Returned with feedback, 2017-03-24) - judged not relevant
- P13: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P14: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1645/) (cf, Returned with feedback, 2018-10-01) - judged not relevant
- P15: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P16: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P17: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P18: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P19: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P20: [COPY FREEZE and PD_ALL_VISIBLE](https://commitfest.postgresql.org/patch/401/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P21: [Allow tuple visibility checks without hint-bit maintenance](https://commitfest.postgresql.org/patch/7212/) (cf, Withdrawn, 2026-08-30) - judged not relevant
- P22: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant
- P23: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P24: [Revert "Still further tweaking of deadlock isolation tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3992188c2a8702bcb92140a840b5378b27468921) (revert, Reverted, 2016-02-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 99 s; cost reported by the CLI: $0.88
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 49 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:42:08.530Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
