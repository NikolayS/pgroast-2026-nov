# Design pre-review: Historic snapshot doesn't track txns committed in BUILDING_SNAPSHOT state

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5029](https://commitfest.postgresql.org/patch/5029/) (Needs review); authors: ChangAo Chen (chenchangao)
- Thread: [Historic snapshot doesn't track txns committed in BUILDING_SNAPSHOT state](https://www.postgresql.org/message-id/flat/tencent_6AAF072A7623A11A85C0B5FD290232467808%40qq.com), started 2024-06-09
- Messages read: 10 of 10
- Patch prose read: v3-0001-Track-transactions-committed-in-BUILDING_SNAPSHOT.patch, v3-0002-Add-test-case-snapshot_build-for-test_decoding.patch, v1-0001-Track-transactions-committed-in-BUILDING_SNAPSHOT.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal: message #2 agrees this is a bug, and the fix adds no user-visible surface. The one surviving objection, maintenance-fit-1, is a fixable design point about a separate list of catalog-marking WAL records in the BUILDING_SNAPSHOT path. The visible material does not show v4-0001, so it is unknown whether that point is already answered. What remains is a correctness judgment in the snapshot builder that needs a committer who knows logical decoding.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code and v4-0001 were not read, so the correctness of tracking commits before FULL_SNAPSHOT is not judged. It was also not checked whether snapshot files in pg_logical/snapshots, written by unfixed binaries, keep the bad state after a minor upgrade.

## The proposal, restated

**Problem.** Logical decoding does not record transactions that commit while the snapshot builder is in BUILDING_SNAPSHOT state, because xact_decode() returns early before FULL_SNAPSHOT. A later transaction can then get a historic snapshot that misses a catalog change, and decoding fails with "could not map filenumber ... to relation OID". This breaks logical replication or slot creation for users who create a subscription or slot while other transactions run DDL.

**Proposed design.** The patch makes decoding track transactions committed during BUILDING_SNAPSHOT so that later snapshots see their catalog changes. Since the historic snapshot tracks only catalog-modifying transactions, it also handles WAL records that may mark a transaction as catalog modifying during BUILDING_SNAPSHOT, namely XLOG_HEAP2_NEW_CID and XLOG_HEAP_INPLACE. A second patch adds an isolation test, snapshot_build, to contrib/test_decoding.

**What the author claims.**
- The bug can make a transaction take an incorrect historic snapshot and stop logical replication.
- v1 passes check-world.
- The new snapshot_build test in test_decoding reproduces the scenario.

**Thread state.** Michael Paquier agreed it is a bug, and Bertrand Drouvot backed the approach, reviewed v2 and v3, and found v3-0002 fine. Ajin Cherian asked why the patch calls ReorderBufferXidSetCatalogChanges instead of SnapBuildProcessNewCid, and the author points to v4-0001 in one of three parallel threads.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-06-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Hand-kept list of catalog-marking WAL records in the BUILDING_SNAPSHOT path

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby, backpatchability · `maintenance-fit-1`

A later patch adds or changes a WAL record that marks a transaction as catalog modifying and updates the FULL_SNAPSHOT decode path only. Slots created while such transactions commit get a snapshot that misses the change. Users see the same "could not map filenumber" error, and no test or assertion catches the gap.

v3-0001 adds a second, state-specific branch in decode.c that handles only XLOG_HEAP2_NEW_CID and XLOG_HEAP_INPLACE, with the comment "Currently only XLOG_HEAP2_NEW_CID means a catalog modifying". Message #8 suggests routing BUILDING_SNAPSHOT through the existing record handlers, so that future changes there apply here too. Message #9 asks why the patch calls ReorderBufferXidSetCatalogChanges and does not use SnapBuildProcessNewCid. The visible material does not show that v4-0001 removes the duplicated list or answers either point.

Thread: [#8 (Bertrand Drouvot, 2024-08-12)](https://www.postgresql.org/message-id/Zrmh7X8jYCbFYXjH%40ip-10-97-1-34.eu-west-3.compute.internal), [#9 (Ajin Cherian, 2025-03-26)](https://www.postgresql.org/message-id/CAFPTHDZaKuS8j72C5s63Hisq6ZRCUu4%2BxTRSc31_vHgRhaHxzg%40mail.gmail.com), [#10 (cca5507, 2025-03-26)](https://www.postgresql.org/message-id/tencent_358CCA18E3191FF8D955865A17F6D49EA50A%40qq.com)

**Question:** Does v4-0001 reuse the existing record handlers so that a new catalog-marking record is covered without a separate BUILDING_SNAPSHOT edit, and if not, what catches the omission?

Counter-check: held. Message #8 raises the same concern about the v3 branch in decode.c, and message #9 asks a related question. The author's reply points to v4-0001 in another thread, which the visible material does not show, so nothing here answers the point.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes how decode.c handles commit, XLOG_HEAP2_NEW_CID and XLOG_HEAP_INPLACE records before FULL_SNAPSHOT. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | This is a bug fix in logical decoding, and the thread does not say which branches receive it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v3-0001 adds a BUILDING_SNAPSHOT branch with its own list of catalog-marking records that later patches must keep in sync. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v4-0001 reuse the existing record handlers during BUILDING_SNAPSHOT, and if not, what catches a new catalog-marking record that is added only to the FULL_SNAPSHOT path?
2. Why does the patch call ReorderBufferXidSetCatalogChanges for XLOG_HEAP2_NEW_CID and not SnapBuildProcessNewCid, as message #9 asks?
3. Which of the three parallel threads holds the current patch, and can the review continue in one thread?
4. Which branches should receive the fix, and does the patch apply to them without changes that affect extensions?
5. Why did v1 change snapbuild.c while v3 changes only decode.c?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the snapshot builder and the reorder buffer in logical decoding. Masahiko Sawada is listed as a reviewer on the CommitFest entry but has not posted in the visible thread.
- **Liability:** The owner signs for the correctness of historic snapshots built during slot creation on every branch that receives the fix. A mistake here causes decoding errors or wrong decoded data, with no time limit.
- **Author follow-up:** The author posted v1 to v4, answered reviewers, and moved the entry across CommitFests for more than two years, so the author is likely to stay for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug is agreed in message #2 and the patch is small, but the entry has been in 12 CommitFests with no committer posting on the design. The patch is small, but snapshot builder state changes need careful review, and the work is split across three threads.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this fix. Message #10 points to three parallel threads on this bug, with v4-0001 in one of them, but the material does not show those threads or any rejection reason from them.
- standards-compat: 0 objection(s). This is an internal bug fix in logical decoding with no user-facing surface. It adds no SQL syntax, GUC, catalog object, psql change or dump content, so nothing is left for a standards or compatibility objection to rest on. Back-patch scope and any change in which transactions get decoded right after slot creation could not be checked without v4-0001.
- replication-ondisk: 0 objection(s). The fix changes in-memory snapshot builder state in decode.c and adds no WAL record, catalog, wire or ABI change. One point could not be checked: whether snapshot files already in pg_logical/snapshots, written by unfixed binaries, keep the bad state after a minor upgrade.
- security: 0 objection(s). The proposal fixes a correctness bug inside logical decoding's snapshot builder and adds a test. It does not change who can create slots or subscriptions, adds no privilege, role, setting or interface, and exposes no new data, so there is no security or privilege aspect to object to.
- maintenance-fit: 1 objection(s). The fix is desirable, and its cost falls only on decoding during slot creation. From this angle the only concern is the duplicated record list. Committability: Masahiko Sawada is listed as a reviewer, but no committer shows engagement in the visible thread, and the work is split across three threads.
- already-exists: 0 objection(s). This is a bug fix in decode.c for an error users should never see, as message #2 says. Postgres has no existing feature, option, extension or external tool that avoids the wrong historic snapshot, so nothing from the already-exists angle applies.
- demand: 0 objection(s). This is a bug fix for a user-visible error ("could not map filenumber ... to relation OID") that stops logical replication when a slot or subscription is created during concurrent DDL. A committer agreed it must be fixed (message #2), so demand is not the open question.

### Precedents retrieved

- P1: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P2: [Global snapshots](https://commitfest.postgresql.org/patch/1662/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P3: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P4: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant; reason summarised from its thread
- P5: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant; reason summarised from its thread
- P6: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P7: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant; reason summarised from its thread
- P8: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P9: [snapshot too old, configured by time](https://commitfest.postgresql.org/patch/348/) (cf, Returned with feedback, 2016-02-01) - judged not relevant
- P10: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P11: [Support logical replication of DDL commands](https://commitfest.postgresql.org/patch/3595/) (cf, Withdrawn, 2024-03-28) - judged not relevant; reason summarised from its thread
- P12: [Allow "snapshot too old" error, to prevent bloat](https://commitfest.postgresql.org/patch/150/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P13: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P14: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P16: [Revert "Modified files for MERGE"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7cf8a5c302735d193dcf901b87e03e324903c632) (revert, Reverted, 2018-04-02) - judged not relevant
- P17: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - judged not relevant
- P18: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P19: [Fix pg_stat_statements query jumbling to use relation name instead of relation oid](https://commitfest.postgresql.org/patch/365/) (cf, Rejected, 2015-11-15) - judged not relevant
- P20: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P21: [Allow INSTEAD OF DELETE triggers to modify the tuple for RETURNING](https://commitfest.postgresql.org/patch/1237/) (cf, Returned with feedback, 2017-09-19) - judged not relevant
- P22: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P23: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P24: [Revert no-op changes to BufferGetPage()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a343e223a5c33a7283a6d8b255c9dbc48dbc5061) (revert, Reverted, 2016-04-20) - judged not relevant

### Warnings

- CommitFest entry has 3 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/tencent_6AAF072A7623A11A85C0B5FD290232467808%40qq.com). Others: https://www.postgresql.org/message-id/flat/tencent_8DEC9842690A9B6AFD52D4659EF0700E9409%40qq.com, https://www.postgresql.org/message-id/flat/tencent_FA60D4EE3E14ACF0B936396551260A4FFD05%40qq.com

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 152 s; cost reported by the CLI: $1.37
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 46 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:29:07.490Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
