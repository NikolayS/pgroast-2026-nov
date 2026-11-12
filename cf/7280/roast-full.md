# Design pre-review: Use maintenance_io_concurrency for parallel index builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7280](https://commitfest.postgresql.org/patch/7280/) (Needs review); authors: Jingtang Zhang (mrdrivingduck)
- Thread: [\[PATCH\] Use maintenance_io_concurrency for parallel index builds](https://www.postgresql.org/message-id/flat/CAPsk3_BaXgiA3c%3DGR63kU6O478MWggYtBCx3vWnW6d87F3%3D0nA%40mail.gmail.com), started 2026-09-08
- Messages read: 5 of 5
- Patch prose read: v4-index-maintenance-io.patch, v1-parallel-index-maintenance-io.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: these scans do maintenance work and reuse an existing flag and setting. Two fixable points remain. standards-compat-1 asks why a change in prefetch depth belongs in stable branches, and which ones. maintenance-fit-2 asks for a rule on which scans pass SO_MAINTENANCE and why the concurrent validation scan is left out. A committer can settle both, most likely by applying to master only.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can confirm that the tableam.h change is ABI-compatible on stable branches or that every maintenance heap scan now passes the flag.

## The proposal, restated

**Problem.** Heap scans in index builds and table rewrites take their read stream prefetch depth from effective_io_concurrency, a query setting, so maintenance_io_concurrency has no effect on them. This affects administrators who tune maintenance I/O separately. The CommitFest entry tags it as a bugfix targeting stable branches.

**Proposed design.** The patch passes the maintenance context (SO_MAINTENANCE) through existing scan flags so these read streams use maintenance_io_concurrency, including tablespace overrides. Serial builds call table_beginscan_common() directly instead of changing the table AM API. v4 covers serial and parallel B-tree, GIN and BRIN builds, BRIN range summarization, and the sequential heap scan of VACUUM FULL, CLUSTER and REPACK.

**User-visible surface.**
- Behaviour change: maintenance_io_concurrency (and its tablespace override) now controls heap prefetch in create index, BRIN summarization, VACUUM FULL, CLUSTER and REPACK.
- src/include/access/tableam.h changes, which table AM extensions may see; the commit message says interfaces are preserved.

**What the author claims.**
- Parallel B-tree, GIN and BRIN builds use effective_io_concurrency because they do not pass READ_STREAM_MAINTENANCE.
- The fix keeps existing scan options and interfaces unchanged.
- The heap validation scan of concurrent index creation stays unchanged on purpose.
- Yuhang Qiu agrees the problem is real and says the fix looks good.

**Thread state.** Yuhang Qiu reviewed v1 and suggested the serial-path approach, which v3 adopted. Nobody has replied to v4, which added the VACUUM FULL, CLUSTER and REPACK scans.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatching changes prefetch depth for existing installations on a minor upgrade

fixable · level 1b, rubric item 6 · backpatchability · `standards-compat-1`

A DBA raised effective_io_concurrency for fast storage and left maintenance_io_concurrency at its default. After a minor upgrade, create index, VACUUM FULL and CLUSTER prefetch less, and nightly maintenance windows run longer. The DBA learns of it only when jobs overrun, with no release note telling them which setting to change.

The CommitFest entry targets "stable", but none of the five messages argues for a backpatch or says which branches. The patch moves which setting controls I/O for common maintenance commands, so tuned systems see a change they did not ask for in a minor release. A committer would likely treat this as a master-only behaviour change unless the docs already promise the new behaviour. REPACK also appears in v4, so a backpatched version would differ by branch and still needs to be defined.

Thread: [#1 (Jingtang Zhang, 2026-09-08)](https://www.postgresql.org/message-id/CAPsk3_BaXgiA3c%3DGR63kU6O478MWggYtBCx3vWnW6d87F3%3D0nA%40mail.gmail.com), [#5 (Jingtang Zhang, 2026-10-28)](https://www.postgresql.org/message-id/CAPsk3_ASDKa-3fhMTeXqPoa%2BEV35S26_%2BP7uXN_75F_jcbzu0A%40mail.gmail.com)

**Question:** Which branches should get this, and why is a changed I/O setting for create index and VACUUM FULL acceptable in a minor release for already tuned systems?

Counter-check: held. The CommitFest entry targets stable, but no message names branches or argues for a backpatch. The scenario is concrete: a site that raised only effective_io_concurrency gets shallower prefetch for create index and VACUUM FULL after a minor upgrade.

### 2. No stated rule for which scans count as maintenance

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A developer later adds a heap scan to a maintenance command, or a table AM implements index_build_range_scan, without passing SO_MAINTENANCE. That scan silently uses effective_io_concurrency, and nothing fails. Users see one command follow two settings and report it as a new bug.

Coverage is opt-in per call site. v4 shows this, because the copy_for_cluster scan was found weeks after v3 (message #5). Create index concurrently would also still mix two settings in one command, since the validation scan is left unchanged without a stated reason. A committer would likely ask for a written rule, in the docs or near the flag, that says which scans must pass it.

Thread: [#3 (Yuhang Qiu, 2026-09-09)](https://www.postgresql.org/message-id/097AEAB9-FD3E-4B58-922D-8F5AFC06A709%40gmail.com), [#5 (Jingtang Zhang, 2026-10-28)](https://www.postgresql.org/message-id/CAPsk3_ASDKa-3fhMTeXqPoa%2BEV35S26_%2BP7uXN_75F_jcbzu0A%40mail.gmail.com)

**Question:** What rule decides which scans must pass SO_MAINTENANCE, and why does the concurrent index validation scan fall outside it?

Counter-check: held. Per-call-site opt-in copies how READ_STREAM_MAINTENANCE already works, so that part is not this patch's fault. The commit message still gives no reason for leaving the concurrent validation scan unchanged, and message #5 shows a scan was missed without a stated rule.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes src/include/access/tableam.h, which out-of-tree table AMs compile against, and the entry targets stable branches. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but maintenance_io_concurrency gains control of create index, BRIN summarization, VACUUM FULL, CLUSTER and REPACK heap scans. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry is tagged Bugfix with target version stable, and REPACK does not exist in all branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each maintenance scan must pass SO_MAINTENANCE itself, and nothing fails when a new call site omits it. |
| performance on the hot path | **yes** | The patch changes the prefetch depth of heap scans in index builds and table rewrites, with no benchmark shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which stable branches should get this change, and why is a different prefetch depth for create index and VACUUM FULL acceptable in a minor release?
2. What rule decides which scans must pass SO_MAINTENANCE, and why does the heap validation scan of concurrent index creation fall outside it?
3. Does the tableam.h change keep struct layout and flag values compatible for table AM extensions compiled against current stable branches?
4. What is the measured effect on create index and VACUUM FULL time when maintenance_io_concurrency and effective_io_concurrency differ?
5. Are non-heap table AMs expected to honour SO_MAINTENANCE in their own index build and rewrite scans?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the read stream code and the table AM scan interface would have to own it; no committer has spoken in the visible thread.
- **Liability:** The owner signs for a change in maintenance I/O depth on tuned systems, for tableam.h compatibility if backpatched, and for later reports of scans that still use effective_io_concurrency.
- **Author follow-up:** The author posted four versions in the thread and adopted the reviewer's serial-path suggestion in v3, which suggests continued engagement.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 5.7 KB, reuses an existing flag and setting, and helps administrators who tune maintenance I/O separately. The open points are the backpatch scope and a stated rule for which scans pass the flag.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Documented contract of maintenance_io_concurrency is not updated or made consistent. Refuted (not specific): Duplicate of maintenance-fit-2, which asks the same question about the rule and the concurrent validation scan and has thread evidence from message #5. The remaining part is the missing documentation change, which the rubric puts out of scope at level 4.
- `maintenance-fit-1` Backpatching changes index build and rewrite I/O in minor releases. Refuted (not specific): Duplicate of standards-compat-1, which has the same scenario and the same question and also covers the per-branch difference for REPACK.
- `demand-1` No user report to justify changing I/O behaviour in stable branches. Refuted (not specific): Duplicate of standards-compat-1: the objection accepts the change for master and its failure scenario is the same backpatch regression. The missing field report adds no separate failure beyond the backpatch question.

### Review angles

- prior-attempts: 0 objection(s). The precedent search found no earlier proposal with the same or a similar idea, and no thread message points to an earlier discussion. Without a prior rejection reason to apply, the prior-attempts angle has no objection to raise.
- standards-compat: 2 objection(s). Both objections concern scope and release policy. Moving these scans to maintenance_io_concurrency looks consistent with what the setting is meant for.
- replication-ondisk: 0 objection(s). The patch changes only the prefetch depth of read streams in index builds and table rewrites. It adds no on-disk format, WAL record, catalog change, wire protocol message or cross-node state, so standbys, subscribers and pg_upgrade are unaffected. The nbtree VACUUM bookkeeping invariant does not apply. The one open point is the tableam.h change in a stable-branch target: a committer would likely ask whether the struct and flag layout stay ABI-compatible for out-of-tree table AMs, which cannot be checked without the code.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It only changes which existing setting, effective_io_concurrency or maintenance_io_concurrency, sets the prefetch depth for scans the user is already allowed to run; both settings are user-settable today with the same tablespace overrides, so nothing new becomes visible or reachable.
- maintenance-fit: 2 objection(s). The core idea is small and reuses an existing mechanism, so I see no fit or hot-path objection. The nbtree VACUUM bookkeeping invariant does not apply, because the patch changes only prefetch depth.
- already-exists: 0 objection(s). The proposal adds no new mechanism. It makes the existing maintenance_io_concurrency and READ_STREAM_MAINTENANCE path cover more maintenance scans. Running `set effective_io_concurrency` before create index or VACUUM FULL gets the same effect, but that works around the mismatch rather than offering a competing feature, so no already-exists objection applies.
- demand: 1 objection(s). The change itself fits the documented purpose of maintenance_io_concurrency and costs little for non-users. A master-only version would draw no demand objection from this angle.

### Precedents retrieved

- P1: [Index Insert Prefetching](https://commitfest.postgresql.org/patch/4622/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P2: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant; reason summarised from its thread
- P4: [Fix search_path for maintenance commands](https://commitfest.postgresql.org/patch/4367/) (cf, Withdrawn, 2024-01-18) - judged not relevant; reason summarised from its thread
- P5: [Revert "Fix search_path to a safe value during maintenance operations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2fcc7ee7affe2fb708d7b2727e93981216b5d2f8) (revert, Reverted, 2023-06-10) - judged not relevant
- P6: [Allow tuple visibility checks without hint-bit maintenance](https://commitfest.postgresql.org/patch/7212/) (cf, Withdrawn, 2026-08-30) - judged not relevant; reason summarised from its thread
- P7: [Allow tuple visibility checks without hint-bit, maintenance](https://commitfest.postgresql.org/patch/7215/) (cf, Withdrawn, 2026-08-29) - judged not relevant; reason summarised from its thread
- P8: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P9: [Revert "Optimize pg_visibility with read streams."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ddfc556a644404a8942e77651f75f09aa5188782) (revert, Reverted, 2024-09-04) - judged not relevant
- P10: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P11: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P12: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P13: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P14: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P15: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant
- P16: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P17: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P18: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant
- P19: [BRIN bloom and multi-minmax indexes](https://commitfest.postgresql.org/patch/1348/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P20: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P21: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant
- P22: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant
- P24: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 126 s; cost reported by the CLI: $0.87
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 69 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:36:33.814Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
