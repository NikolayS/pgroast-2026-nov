# Design pre-review: Improve hash aggregate spilling by writing only the needed columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7310](https://commitfest.postgresql.org/patch/7310/) (Needs review); authors: Mario Karuza (mkaruza)
- Thread: [Improve hash aggregate spilling by writing only the needed columns](https://www.postgresql.org/message-id/flat/7247cfafe659f49340ebd5d9e31a5ced45381879.camel%40icloud.com), started 2026-09-16
- Messages read: 5 of 5
- Patch prose read: v2-0001-Write-only-the-needed-columns-to-hash-aggregate-s.patch, v1-0001-Write-only-the-needed-columns-to-hash-aggregate-s.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem and the design are stated in message #1, the cost falls only on executions that spill, and two reviewers reproduced the gains. The earlier planner-side attempt [P3] was reverted for a regression that this executor-side design avoids. Confidence is medium because the code was not read and no committer has spoken in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the handling of input slot types, Datum ownership between the compact slot and the input slot, and memory context use on rescan could not be checked. Test coverage in v2 is unknown.

## The proposal, restated

**Problem.** Hash aggregate spills tuples using the full outer plan tuple descriptor, with unneeded columns set to NULL. When a seq scan passes whole table tuples, wide tables make the aggregate write and read much more spill data than it uses. This affects analytic queries such as TPC-H Q18.

**Proposed design.** On the first spill, nodeAgg builds a compact tuple descriptor with only the needed columns and a map back to the input layout. The write path gathers needed values into the compact slot, and the read path scatters them back into the input tuple layout. A tuple read from a spill file that must spill again is written as read, without deform and form.

**What the author claims.**
- TPC-H Q18 at SF5 and SF10 runs about 24% faster and spills about 21-23% less, with work_mem = 4MB.
- No behaviour change when all input columns are needed.
- No cost for executions that never spill, after the deferred initialization in v2.
- Reviewer Alain Lesage reports a -38% geometric mean on the 14 spilling ClickBench queries.

**Thread state.** Three reviewers replied. Xiaoxuan Li found a JIT crash with grouping sets from a stale FIELDNO_AGGSTATE_ALL_PERGROUPS, fixed in v2 together with the deferred initialization that Ethan Tran suggested, and Alain Lesage confirmed both fixes and the gains on v2.

## 2. Prior attempts

### Revert "Use CP_SMALL_TLIST for hash aggregate" (similar idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0babd109801e5ecd90df29589c23c6daf3ae69f7](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0babd109801e5ecd90df29589c23c6daf3ae69f7)
- **Thread:** [https://www.postgresql.org/message-id/flat/20200614181418.mx4bvljmfkkhoqzl@alap3.anarazel.de](https://www.postgresql.org/message-id/flat/20200614181418.mx4bvljmfkkhoqzl@alap3.anarazel.de), [https://www.postgresql.org/message-id/flat/20200519151202.u2p2gpiawoaznsv2@development](https://www.postgresql.org/message-id/flat/20200519151202.u2p2gpiawoaznsv2@development)
- **Date:** decided 2020-07-13; started 2020-05-31
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 4cad2534da6d17067d98cf04be2dfc1bda8f2cd0 due to a performance regression. It will be replaced by a new approach in an upcoming commit.
- **Reason (quote):** "This reverts commit 4cad2534da6d17067d98cf04be2dfc1bda8f2cd0 due to a performance regression."
- **What is different this time:** P3 narrowed the hash aggregate input in the planner with CP_SMALL_TLIST, so every hash aggregate paid for the projection. This patch leaves the planner and the scan unchanged and narrows tuples only inside nodeAgg, on the first spill.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds fields to AggState in execnodes.h, which shifts struct offsets and the JIT field number FIELDNO_AGGSTATE_ALL_PERGROUPS. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | nodeAgg.c gains a second spill tuple layout, a column map and a direct respill path, with no regression tests so far. |
| performance on the hot path | **yes** | The patch changes spill code reached from lookup_hash_entries(), which runs for every input tuple; hashagg_spill_tuple() is marked pg_noinline for that reason. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v2 add regression tests for sparse-column reconstruction, respill, grouping sets with spilling, rescans and the JIT case from message #3?
2. How does the write path handle an input slot that is neither a heap tuple slot nor a minimal tuple slot?
3. How long do the Datums that the reconstructed input slot borrows from the compact spill slot stay valid, and where does the code document that?
4. Will the commit message say that queries grouping on expressions see no gain because the scan already projects?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the executor's hash aggregate spill code in nodeAgg.c and the JIT type synchronization rule for AggState.
- **Liability:** The owner signs for the correctness of the compact spill layout, the column map and the direct respill path in every later release. Spill files are temporary, so there is no on-disk or upgrade commitment.
- **Author follow-up:** The author posted v2 with the JIT fix from message #3 and the deferred initialization from message #2, which shows response to review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 10 KiB in two files, adds no user-visible surface, and shows about 24% on TPC-H Q18 and a reviewer-reported -38% geometric mean on spilling ClickBench queries. It helps analytic queries on wide tables that spill, and it needs regression tests and a committer who knows nodeAgg.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The only precedent is [P3], reverted for a performance regression. In that design every hash aggregate projected in the planner, so all of them paid, not only the ones that spill. This patch puts the cost only on the spill path, and messages #2 and #5 measured no overhead when nothing spills. Message #1 cites two earlier discussions of the same goal. The thread reports that they led to today's behaviour of setting unneeded columns to NULL, and the proposal does not say the new approach repeats what failed there.
- standards-compat: 0 objection(s). The patch changes only the private spill file format inside nodeAgg. It adds no SQL syntax, GUC, catalog object, psql behaviour or dump content. Spill files are temporary and never cross an upgrade or reach a client. Query results and plans stay the same, and EXPLAIN output changes only in reported disk usage. From the standards and compatibility angle, there is no user-facing contract to object to. [P3] was a planner-side tlist change reverted for a performance regression, so its recorded reason does not apply here.
- replication-ondisk: 0 objection(s). The patch changes only the format of temporary hash aggregate spill files, which live for one query in one backend and are never read after a crash, by a standby, by pg_upgrade or by another node, so there is no on-disk, WAL, catalog or wire protocol concern. The new AggState field in execnodes.h shifts struct offsets, but this is a master-only change; the JIT field number was fixed in v2 (message #4) and confirmed in message #5.
- security: 0 objection(s). The proposal changes only the internal format of hash aggregate spill files inside one executor node. It adds no SQL surface, privilege, setting or monitoring output, and spill files stay private to the backend, so there is no new privilege or exposure aspect to object to. Spill disk use goes down, so the existing temp_file_limit and work_mem limits on disk use by unprivileged users still apply as before.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection to raise. The change stays inside nodeAgg.c, spill files are temporary with no on-disk or upgrade commitment, there is no user-visible surface, and v2 measured no cost when nothing spills (messages #2, #4, #5). [P3] reverted a planner-side approach for a regression; this executor-side design avoids that path. Committability notes: no committer has engaged in the visible thread. The JIT field-number trap in message #3 is an existing rule, but the absence of regression tests leaves that trap and the respill path without coverage.
- already-exists: 0 objection(s). No existing feature, setting or extension writes only the needed columns to hash aggregate spill files, and users cannot make a plain seq scan project for the aggregate. The patch extends the current NULL-out spill path in nodeAgg; the planner-side alternative was reverted for a regression [P3], and this proposal stays inside the executor.
- demand: 0 objection(s). The demand looks real. Wide-table analytic aggregates that spill are common, and the gains show up on TPC-H Q18 and on 14 spilling ClickBench queries reproduced independently in message #5. The change adds no user-visible surface, and the cost is confined to nodeAgg spill code.

### Precedents retrieved

- P1: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P2: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - judged not relevant
- P3: [Revert "Use CP_SMALL_TLIST for hash aggregate"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0babd109801e5ecd90df29589c23c6daf3ae69f7) (revert, Reverted, 2020-07-13) - used in the review
- P4: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant; reason summarised from its thread
- P5: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant; reason summarised from its thread
- P6: [Memory-bounded HashAgg](https://commitfest.postgresql.org/patch/63/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P7: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P8: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P9: [Memory accounting towards memory-bounded HashAgg](https://commitfest.postgresql.org/patch/59/) (cf, Rejected, 2015-01-16) - judged not relevant; reason summarised from its thread
- P10: [Allow build.pl just to construct project files without running a build](https://commitfest.postgresql.org/patch/1668/) (cf, Rejected, 2018-07-12) - judged not relevant; reason summarised from its thread
- P11: [Memory Accounting (towards memory-bounded HashAgg)](https://commitfest.postgresql.org/patch/269/) (cf, Rejected, 2015-07-26) - judged not relevant; reason summarised from its thread
- P12: [onlyvalue aggregate](https://commitfest.postgresql.org/patch/407/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P13: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P14: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P15: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant
- P16: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P17: [Reducing serialized size of Nodes from nodeToString output](https://commitfest.postgresql.org/patch/4685/) (cf, Returned with feedback, 2024-04-04) - judged not relevant
- P18: [Reduce size of fmgr_builtins array by 1/3](https://commitfest.postgresql.org/patch/2411/) (cf, Rejected, 2020-01-18) - judged not relevant
- P19: [Flush disk write caches by default on macOS and Windows](https://commitfest.postgresql.org/patch/4453/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P20: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P21: [Handling of the input data errors in COPY FROM](https://commitfest.postgresql.org/patch/1179/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P22: [Index-only filters without IOS](https://commitfest.postgresql.org/patch/4352/) (cf, Withdrawn, 2024-06-21) - judged not relevant
- P23: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P24: [Add Hook Functions for Disk Quota Extension](https://commitfest.postgresql.org/patch/1883/) (cf, Returned with feedback, 2020-07-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 87 s; cost reported by the CLI: $1.19
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 42 text field(s) checked, 4 finding(s), 4 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:39:44.490Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
