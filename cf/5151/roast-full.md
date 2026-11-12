# Design pre-review: DirtySnapshot index scan skips concurrently updated tuples leading to lost updates/deletes in logical replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5151](https://commitfest.postgresql.org/patch/5151/) (Needs review); authors: Michail Nikolaev (nkey), Mihail Nikalayeu (mihailnikalayeu)
- Thread: [\[BUG?\] check_exclusion_or_unique_constraint false negative](https://www.postgresql.org/message-id/flat/CANtu0oiktqQ2pwExoXqDpByXNCJa-KE5vQRodTRnmFHN_%2BqwHg%40mail.gmail.com), started 2024-07-14
- Messages read: 45 of 45
- Patch prose read: v4-0001-Fix-possible-lost-tuples-in-non-MVCC-index-scans-.patch, v4-0002-Add-isolation-test-to-reproduce-dirty-snapshot-sc.patch, test_+_assert_to_reproduce_possible_issue_with_check_exclusion_or_unique_constraint.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

The problem is stated and reproduced by two other people, so levels 1a and 1b pass. Two fixable level 2 objections survive, replication-ondisk-1 and standards-compat-1, both about the in-flight local INSERT case measured in message #44. A committer who knows the apply worker must decide whether that behaviour change is acceptable, in particular on stable branches. The v20 patch and its discussion are in a thread not shown here, so confidence is low.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v20 patch, its thread and the related documentation thread are not shown, and no code was read. The author's reasons for declining the variant in message #44 could not be checked, nor could the cost of the extra snapshot.

## The proposal, restated

**Problem.** A btree index scan with SnapshotDirty or SnapshotSelf can return no row when a concurrent non-HOT update of that row puts the new index entry on the same page, or on a page to the left. RelationFindReplTupleByIndex uses such a scan, so the subscriber apply worker can lose publisher updates or deletes and report the wrong conflict type.

**Proposed design.** The current version (v11 to v20) scans with a fresh MVCC snapshot from GetLatestSnapshot before each attempt in the logical replication tuple lookup, and table_tuple_lock still waits for concurrent updaters. Earlier versions tracked max(xmax) and rescanned (v1 to v4, dropped by the author as unsound), then held btree page read locks longer (v5 to v10). A test patch adds injection points and TAP tests.

**User-visible surface.**
- Logical replication conflict reports (update_missing, delete_missing, update_origin_differs, update_deleted) change for concurrent subscriber updates.
- An in-flight local INSERT of the same key is no longer waited for, so the apply worker reports update_missing and skips the publisher UPDATE.
- New injection points for tests, now in heapam_index_getnext_slot().

**What the author claims.**
- The race exists in every release since logical replication and since the first UPSERT commit.
- Concurrent subscriber updates, even to subscriber-only columns, can make publisher updates or deletes get lost.
- The MVCC scan gives the same guarantees as SnapshotDirty, without missed tuples and with the same performance.
- Master only wins the in-flight INSERT race by chance, so v20 leaves the wait out.

**Thread state.** Amit Kapila questioned whether SnapshotDirty results are wrong and suggested documenting the race; Manu rebased to v19, confirmed the tests fail without the fix, and showed an in-flight INSERT regression with a variant that keeps the wait. The author declined the variant and moved v20 to a separate thread, not shown here.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-07-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. MVCC lookup drops the wait for an in-flight local INSERT

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-1`

A subscriber session has an uncommitted INSERT of key 1 when the publisher's UPDATE of key 1 arrives. Under v19/v20 the apply worker does not wait, so it logs update_missing and skips the UPDATE. The subscriber keeps 'fromsub' while the publisher has 'frompubnew', and the user only learns this from the conflict log or from diverged data.

Message #44 measured this five times: master waits, applies the UPDATE and reports no conflict, while the patch loses the UPDATE. In message #45 the author says master only wins this race by chance, so v20 leaves the wait out. A committer would likely see a lost update moved from one case to another, because the conflict-detection code relies on the wait. Manu's variant keeps the wait and does not bring the concurrent-update race back, and the material gives no reason why it is unsound.

Thread: [#36 (Mihail Nikalayeu, 2025-09-03)](https://www.postgresql.org/message-id/CADzfLwVj9ogzrf2P_H9Xb1z8vLEzBemBxp1nC9wCg4KaOFbvmw%40mail.gmail.com), [#44 (Manu, 2026-09-22)](https://www.postgresql.org/message-id/179010806896.909920.249094834696255448%40gmail.com), [#45 (Mihail Nikalayeu, 2026-10-05)](https://www.postgresql.org/message-id/CADzfLwUKAnaaadNFmVXk%2BED%3DdStL_FOU0jFzbzVPmA0NW-xstA%40mail.gmail.com)

**Question:** Which case does the extra SnapshotDirty pass from message #44 handle incorrectly, and why is losing the in-flight INSERT wait better than keeping it?

Counter-check: held. Message #36 argues only the INSERT versus INSERT case, and message #45 points to another thread for the reasons. The visible material does not say which case the extra SnapshotDirty pass from message #44 handles wrongly.

### 2. Back-patched fix changes the in-flight local INSERT outcome from apply to skip

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `standards-compat-1`

A subscriber application has a local INSERT of key 1 still open when the publisher's UPDATE of key 1 arrives. Today the apply worker waits, then applies the UPDATE. After a minor upgrade it reports update_missing, skips the UPDATE and keeps the local row, and the divergence shows up only when someone compares the two nodes.

Message #44 measured this case in 5 runs on each tree: master waited and applied the UPDATE every time, and v19 skipped it every time. The author answers in message #45 that master only wins a race here, but the measured master behaviour is a lock wait followed by a decision, and that is repeatable. The CF entry targets stable, so users would get this change in a minor release without asking for it. Manu's variant keeps the wait, and the material does not show why it was declined.

Thread: [#44 (Manu, 2026-09-22)](https://www.postgresql.org/message-id/179010806896.909920.249094834696255448%40gmail.com), [#45 (Mihail Nikalayeu, 2026-10-05)](https://www.postgresql.org/message-id/CADzfLwUKAnaaadNFmVXk%2BED%3DdStL_FOU0jFzbzVPmA0NW-xstA%40mail.gmail.com)

**Question:** Why should a stable-branch fix change the outcome for an in-flight local INSERT, when the variant in message #44 keeps master's behaviour and passes the same tests?

Counter-check: held. Message #44 measured the change on both trees, and the CommitFest entry targets stable. The author's reason for declining the variant that keeps the wait is in a thread not shown here, so nothing visible answers the minor-release question.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The fix changes how the logical replication apply worker finds the subscriber row in RelationFindReplTupleByIndex. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and the fix changes conflict outcomes that users of current releases can observe. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The apply worker lookup gets a new snapshot rule, and new injection points sit in heapam_index_getnext_slot(). |
| performance on the hot path | **yes** | The apply worker takes a fresh snapshot with GetLatestSnapshot before each lookup attempt, and no benchmark is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which case does the extra SnapshotDirty pass from message #44 handle incorrectly?
2. Why is skipping the publisher UPDATE with update_missing better than waiting for the in-flight local INSERT, as master does in the five runs of message #44?
3. Which stable branches should get the fix, and how is the in-flight INSERT behaviour change justified in a minor release?
4. What does the fresh MVCC snapshot per lookup attempt cost the apply worker on an update-heavy workload, compared with master?
5. Does the fix leave check_exclusion_or_unique_constraint and other SnapshotDirty callers as they are, and is that intended?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the logical replication apply worker and conflict detection, with a second opinion from someone who knows nbtree scans with non-MVCC snapshots. The brief shows Amit Kapila questioning the premise and suggesting documentation; no committer in the visible material has agreed to the fix.
- **Liability:** The owner signs for the conflict detection results of the apply worker on every branch that gets the fix, including the changed in-flight INSERT outcome. On stable branches this holds until each branch reaches end of life.
- **Author follow-up:** Strong sign that the author stays: more than two years of work, 20 versions, and repeated moves between CommitFests by the author.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The fix prevents lost publisher updates and deletes on subscribers that also take local writes, and the current design is confined to the tuple lookup of the apply worker. The entry has been in 11 CommitFests, and no committer in the visible material has ruled on the in-flight INSERT trade-off or on back-patching.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Back-patch changes subscriber conflict behaviour in minor releases. Refuted (not specific): Duplicate of standards-compat-1. The other parts do not add a failure: corrected conflict types are the purpose of the fix, and where the test injection point lives is an implementation detail outside levels 1-2.
- `maintenance-fit-1` Fix covers one caller and leaves the SnapshotDirty btree limitation for the others. Refuted (no failure scenario): The scenario depends on a future caller that does not exist, and the limitation predates this patch. Messages #4, #13 and #14 explain why the existing callers in check_exclusion_or_unique_constraint are safe through retry and speculative insertion.
- `maintenance-fit-2` Target 'stable' brings a conflict-behaviour change into back branches. Refuted (not specific): Duplicate of standards-compat-1, which states the same in-flight INSERT change in a minor release with the same evidence. The per-branch injection point remark concerns test code, which is outside levels 1-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus returned no earlier attempt at this idea. The only earlier designs are inside this thread: v1 to v4 (dropped by the author in message #18) and v5 to v10 (judged too expensive in message #33), and the current MVCC rescan does not repeat either of them.
- standards-compat: 1 objection(s). From the standards and compatibility angle, the only issue is the behaviour change in the in-flight INSERT case for a stable-branch target. No new syntax, GUC, catalog or pg_dump surface is involved.
- replication-ondisk: 2 objection(s). The current design does not change on-disk format, WAL, catalogs, wire protocol or extension ABI. The nbtree VACUUM bookkeeping invariant does not apply, because v11 and later do not touch nbtree.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes which snapshot the apply worker uses to find a row, adds no new privilege, role, setting or user-callable interface, and changes nothing about what a session can see or do. A local session that could hold an open INSERT to make the apply worker skip a publisher UPDATE already has INSERT privilege on that table and can change the row directly. Master's wait on an open local INSERT already lets such a session stall the apply worker, and v20 removes that wait, so this patch adds no new denial-of-service path. The new injection points exist only in builds made with --enable-injection-points. The visible material does not include the v20 code or thread.
- maintenance-fit: 2 objection(s). The nbtree VACUUM bookkeeping invariant does not apply, because the current design does not change VACUUM. No committer who knows nbtree has engaged in the visible thread. That affects committability, not the verdict.
- already-exists: 0 objection(s). This is a correctness fix inside RelationFindReplTupleByIndex, and no existing feature, setting, contrib module or extension avoids the missed-tuple race in the apply worker. The fix reuses GetLatestSnapshot and table_tuple_lock, which the lookup already calls, so it adds no second mechanism. Documenting the race, as suggested in message #34, would describe the bug without fixing it.
- demand: 0 objection(s). This is a correctness fix, and two other people reproduced the lost update in logical replication (message #13, message #44). A demand objection does not fit. The open questions are design trade-offs, such as the in-flight INSERT case and back-patching, and other angles cover those.

### Precedents retrieved

- P1: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P2: [Global snapshots](https://commitfest.postgresql.org/patch/1662/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P3: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P4: [old_snapshot: add test for coverage](https://commitfest.postgresql.org/patch/3805/) (cf, Returned with feedback, 2022-11-25) - judged not relevant; reason summarised from its thread
- P5: [Revert the commits related to allowing page lock to conflict among parallel group members.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cc32ec24fdf3b9808617439d41b0ba7f3eb16b04) (revert, Reverted, 2023-07-06) - judged not relevant
- P6: [snapshot too old, configured by time](https://commitfest.postgresql.org/patch/348/) (cf, Returned with feedback, 2016-02-01) - judged not relevant
- P7: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P8: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P9: [Allow "snapshot too old" error, to prevent bloat](https://commitfest.postgresql.org/patch/150/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P10: [Avoid duplicate XIDs at recovery when building initial snapshot - take 2](https://commitfest.postgresql.org/patch/1866/) (cf, Returned with feedback, 2018-12-03) - judged not relevant; reason summarised from its thread
- P11: [Time-delayed logical replication subscriber](https://commitfest.postgresql.org/patch/3581/) (cf, Returned with feedback, 2023-03-10) - judged not relevant; reason summarised from its thread
- P12: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P13: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P14: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P15: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/3556/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P16: [Fix missed parallel safety checks and wrong parallel safety](https://commitfest.postgresql.org/patch/3094/) (cf, Withdrawn, 2021-06-21) - judged not relevant
- P17: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P18: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant
- P19: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/4636/) (cf, Withdrawn, 2023-11-01) - judged not relevant
- P20: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant
- P21: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P22: [Improving replay of XLOG_BTREE_VACUUM records](https://commitfest.postgresql.org/patch/364/) (cf, Rejected, 2016-04-03) - judged not relevant
- P23: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P24: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/CANtu0oiktqQ2pwExoXqDpByXNCJa-KE5vQRodTRnmFHN_%2BqwHg%40mail.gmail.com). Others: https://www.postgresql.org/message-id/flat/CADzfLwXZVmbo11tFS_G2i%2B6TfFVwHU4VUUSeoqb%2B8UQfuoJs8A%40mail.gmail.com

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 117 s; cost reported by the CLI: $2.79
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:30:35.668Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
