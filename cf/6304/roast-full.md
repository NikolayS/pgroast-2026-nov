# Design pre-review: Incorrect historic snapshot may be serialized to disk during fast-forwarding

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6304](https://commitfest.postgresql.org/patch/6304/) (Needs review); authors: ChangAo Chen (chenchangao), haiyang li (ocean-li)
- Thread: [\[BUG\] Incorrect historic snapshot may be serialized to disk during fast-forwarding](https://www.postgresql.org/message-id/flat/tencent_3A071B760AA1A38540B57F297332B7781C08%40qq.com), started 2025-11-22
- Messages read: 10 of 10
- Patch prose read: v4-0001-Handle-XLOG_HEAP2_NEW_CID-in-heap2_decode-even-if.patch, v1-0001-Handle-XLOG_HEAP2_NEW_CID-in-heap2_decode-even-if.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated with a reproducer in message #1, and message #10 reports an output plugin failure that v4 fixes. The one surviving objection, replication-ondisk-1, is a fixable level 2 point about snapshot files written before the fix and about back branches. A committer who knows the snapshot builder has to decide those two points.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the snapbuild.c change in v4 and its effect on the serialized snapshot format could not be judged. Only 3 of 10 messages were shown in full; the rest is known from the brief.

## The proposal, restated

**Problem.** During fast-forward decoding, heap2_decode() skips XLOG_HEAP2_NEW_CID, so a transaction that writes only NEW_CID records is not marked as catalog-changing. The serialized historic snapshot then lacks that XID, and a later decoding run at the same LSN builds a different snapshot. The case the thread shows is a write to a user_catalog_table, which matters for output plugins and extensions that scan such tables.

**Proposed design.** heap2_decode() handles XLOG_HEAP2_NEW_CID also when fast-forwarding and marks the transaction as containing catalog changes. The other NEW_CID processing still runs only outside fast-forward. In fast-forward, the record no longer falls through to the XLOG_HEAP2_REWRITE case.

**What the author claims.**
- Two decoding runs at the same LSN produce different historic snapshots, and the incorrect one from the fast-forward run is serialized to disk.
- The bug breaks the snapshot builder design, which tracks all catalogs, even if it may be harmless in practice today.
- An extension that scans a user_catalog_table with the incorrect snapshot gets wrong results.
- The reviewer reproduced an output plugin catalog lookup failure without the patch and confirmed v4 fixes it.
- Common DDL always writes XLOG_XACT_INVALIDATIONS too, so plain DDL does not hit the bug in the reviewer's tests.

**Thread state.** Masahiko Sawada agreed with the analysis but asked for a real-world problem, ideally reproduced with DDL. Yuya Shinde reproduced a plugin failure with a user_catalog_table, found no DDL reproducer, and posted a test patch on top of v4; no committer has replied since.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-11-22; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Wrong snapshot files already on disk keep being reused after the fix

fixable · level 2, rubric item 14 · on-disk format / pg_upgrade, WAL / replication / standby, backpatchability · `replication-ondisk-1`

A user installs a minor release with the fix on a server where an earlier fast-forward run, such as pg_replication_slot_advance, wrote a snapshot file to pg_logical/snapshots without the NEW_CID transaction. A slot created or restarted at that LSN restores the file instead of rebuilding the snapshot. An output plugin that scans a user_catalog_table then reads the wrong rows, with no error to show it.

The patch changes how snapshots are built from now on, but message #1 shows the defect persists as a serialized file that a later run at the same LSN can read. CheckPointSnapBuild removes such files only once no slot needs them. The proposal does not say whether it targets back branches, or whether files written before the fix stay valid, need removal, or need a SNAPBUILD_VERSION bump. The commit message also does not say what the snapbuild.c change does, so a reviewer cannot tell whether the on-disk snapshot format changes.

Thread: [#1 (cca5507, 2025-11-22)](https://www.postgresql.org/message-id/tencent_3A071B760AA1A38540B57F297332B7781C08%40qq.com), [#5 (cca5507, 2025-12-10)](https://www.postgresql.org/message-id/tencent_4B8339BBC550D1C29B5AA1142A1918BCC207%40qq.com), [#10 (Yuya Shinde, 2026-09-03)](https://www.postgresql.org/message-id/CAPgNnN0s7B6jdWT8QBjVA5pkdiPa_zS%2BDO4DgXdXuAp9tWV7fA%40mail.gmail.com)

**Question:** After a minor upgrade, can a snapshot file written by an unpatched fast-forward run still be restored, and if so, what removes or rejects it?

Counter-check: held. A slot that restarts decoding restores the serialized snapshot at its restart point, and the thread does not discuss files written before the fix or back branches. The exposure ends once slots advance and a checkpoint removes the files, so `fixable` is the right severity.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Serialized snapshot files in pg_logical/snapshots written before the fix may be restored after a minor upgrade; v4 also touches snapbuild.c. |
| WAL / replication / standby | **yes** | The patch changes how fast-forward logical decoding in decode.c treats XLOG_HEAP2_NEW_CID records. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but the thread does not say which back branches the fix targets. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. After a minor upgrade, can a snapshot file written by an unpatched fast-forward run still be restored, and if so, what removes or rejects it?
2. Is the fix meant for back branches, and which ones?
3. What does the snapbuild.c change in v4 do, and does it change the serialized snapshot file format?
4. Does any core path or supported extension write NEW_CID without invalidation messages, other than a write to a user_catalog_table?
5. What error or wrong result did the output plugin show in the reproduction in message #10?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the snapshot builder and logical decoding. Masahiko Sawada asked for a real-world problem earlier in the thread, and no committer has replied since the reproduction in message #10.
- **Liability:** The owner signs for historic snapshot correctness on the fast-forward path, in every branch that gets the fix, and for the decision on snapshot files written before it.
- **Author follow-up:** The authors posted revisions up to v4 and moved the entry across CommitFests through 2026-08, so they appear to stay with it.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is under 3 KiB in decode.c and snapbuild.c, adds no user-visible surface, and affects only output plugins and extensions that scan a user_catalog_table. It has been in 5 CommitFests, and no committer has answered the reproduction in message #10.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the retrieved candidates do not cover this idea. Message #1 points to a related historic snapshot bug thread, but the thread says nothing about how it ended, so no earlier rejection reason applies here.
- standards-compat: 0 objection(s). The patch changes only how fast-forward decoding treats XLOG_HEAP2_NEW_CID. It adds no syntax, GUC, catalog object, psql feature or dump content, and it does not change what clients or output plugins are told to expect. Two points could not be checked without the code: whether the snapbuild.c change alters the serialized snapshot file format, and whether wrong snapshots already on disk survive a minor upgrade. The thread does not discuss either point.
- replication-ondisk: 1 objection(s). Only one objection from this angle. The fix adds no WAL record, catalog change or wire protocol change, and touches only snapshot building. The one persistent effect is the serialized snapshot files covered above.
- security: 0 objection(s). The patch adds no privilege, default or new exposure. It changes when fast-forward decoding marks a NEW_CID transaction as catalog-changing, so a serialized historic snapshot includes more XIDs, and the users who can create and advance slots stay the same. The patch has no security or privilege aspect.
- maintenance-fit: 0 objection(s). The fix is a small bugfix in decode.c that lets fast-forward decoding of NEW_CID mark catalog changes, as non-fast-forward decoding already does; it adds no new mechanism, setting or API. The extra cost falls only on fast-forward runs that meet NEW_CID records and matches normal decoding. Not checked without code: what the snapbuild.c hunk in v4 changes. Committability: no committer has engaged since message #6, so the patch has no owner yet.
- already-exists: 0 objection(s). This is a bug fix inside heap2_decode() for fast-forward decoding. No existing feature, option, extension or external tool gives a correct serialized snapshot in this case, and the fix extends the existing NEW_CID handling without adding a second mechanism. Other angles may question demand and real-world impact (message #6).
- demand: 0 objection(s). This is a small correctness fix on the fast-forward path, and message #10 reproduces an output plugin catalog lookup failure through user_catalog_table, the case that option exists for. The thread has no field report, but that affects priority and backpatching rather than whether the fix is wanted, and non-users pay almost nothing.

### Precedents retrieved

- P1: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P2: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P3: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P4: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P5: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P6: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2283/) (cf, Withdrawn, 2020-06-11) - judged not relevant; reason summarised from its thread
- P7: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P8: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Minimal logical decoding on standbys](https://commitfest.postgresql.org/patch/2683/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P10: [Logical decoding of two-phase transactions](https://commitfest.postgresql.org/patch/944/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P11: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P12: [Revert "Improve error message on snapshot import in snapmgr.c"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=be022908cfe2959dbb17c10475deeb492b8e6944) (revert, Reverted, 2023-09-14) - judged not relevant
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P15: [Global snapshots](https://commitfest.postgresql.org/patch/1662/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P16: [Filter irrelevant change before reassemble transactions during logical decoding](https://commitfest.postgresql.org/patch/5585/) (cf, Withdrawn, 2025-08-22) - judged not relevant
- P17: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P18: [XLog size reductions: smaller XLogRecordBlockHeader](https://commitfest.postgresql.org/patch/4324/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P19: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant
- P20: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant
- P21: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant
- P22: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P23: [Filter irrelevant change before reassemble transactions during logical decoding](https://commitfest.postgresql.org/patch/5592/) (cf, Withdrawn, 2025-02-21) - judged not relevant
- P24: [snapshot too old, configured by time](https://commitfest.postgresql.org/patch/348/) (cf, Returned with feedback, 2016-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 81 s; cost reported by the CLI: $0.97
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:47:44.783Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
