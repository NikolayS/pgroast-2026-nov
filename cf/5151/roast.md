# Design pre-review: DirtySnapshot index scan skips concurrently updated tuples leading to lost updates/deletes in logical replication

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5151](https://commitfest.postgresql.org/patch/5151/); [thread](https://www.postgresql.org/message-id/flat/CANtu0oiktqQ2pwExoXqDpByXNCJa-KE5vQRodTRnmFHN_%2BqwHg%40mail.gmail.com), 45 of 45 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). The problem is stated and reproduced by two other people, so levels 1a and 1b pass. Two fixable level 2 objections survive, replication-ondisk-1 and standards-compat-1, both about the in-flight local INSERT case measured in message #44. A committer who knows the apply worker must decide whether that behaviour change is acceptable, in particular on stable branches. The v20 patch and its discussion are in a thread not shown here, so confidence is low.

Not checked: The v20 patch, its thread and the related documentation thread are not shown, and no code was read. The author's reasons for declining the variant in message #44 could not be checked, nor could the cost of the extra snapshot.

## Prior attempts

None found before 2024-07-14.

## Objections a reviewer is likely to raise

### 1. MVCC lookup drops the wait for an in-flight local INSERT

A subscriber session has an uncommitted INSERT of key 1 when the publisher's UPDATE of key 1 arrives. Under v19/v20 the apply worker does not wait, so it logs update_missing and skips the UPDATE. The subscriber keeps 'fromsub' while the publisher has 'frompubnew', and the user only learns this from the conflict log or from diverged data.

Message #44 measured this five times: master waits, applies the UPDATE and reports no conflict, while the patch loses the UPDATE. In message #45 the author says master only wins this race by chance, so v20 leaves the wait out. A committer would likely see a lost update moved from one case to another, because the conflict-detection code relies on the wait. Manu's variant keeps the wait and does not bring the concurrent-update race back, and the material gives no reason why it is unsound.

**Question:** Which case does the extra SnapshotDirty pass from message #44 handle incorrectly, and why is losing the in-flight INSERT wait better than keeping it?

<sub>fixable, level 2, WAL / replication / standby; messages [#36](https://www.postgresql.org/message-id/CADzfLwVj9ogzrf2P_H9Xb1z8vLEzBemBxp1nC9wCg4KaOFbvmw%40mail.gmail.com), [#44](https://www.postgresql.org/message-id/179010806896.909920.249094834696255448%40gmail.com), [#45](https://www.postgresql.org/message-id/CADzfLwUKAnaaadNFmVXk%2BED%3DdStL_FOU0jFzbzVPmA0NW-xstA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Back-patched fix changes the in-flight local INSERT outcome from apply to skip

A subscriber application has a local INSERT of key 1 still open when the publisher's UPDATE of key 1 arrives. Today the apply worker waits, then applies the UPDATE. After a minor upgrade it reports update_missing, skips the UPDATE and keeps the local row, and the divergence shows up only when someone compares the two nodes.

Message #44 measured this case in 5 runs on each tree: master waited and applied the UPDATE every time, and v19 skipped it every time. The author answers in message #45 that master only wins a race here, but the measured master behaviour is a lock wait followed by a decision, and that is repeatable. The CF entry targets stable, so users would get this change in a minor release without asking for it. Manu's variant keeps the wait, and the material does not show why it was declined.

**Question:** Why should a stable-branch fix change the outcome for an in-flight local INSERT, when the variant in message #44 keeps master's behaviour and passes the same tests?

<sub>fixable, level 2, backpatchability; messages [#44](https://www.postgresql.org/message-id/179010806896.909920.249094834696255448%40gmail.com), [#45](https://www.postgresql.org/message-id/CADzfLwUKAnaaadNFmVXk%2BED%3DdStL_FOU0jFzbzVPmA0NW-xstA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which case does the extra SnapshotDirty pass from message #44 handle incorrectly?
2. Why is skipping the publisher UPDATE with update_missing better than waiting for the in-flight local INSERT, as master does in the five runs of message #44?
3. Which stable branches should get the fix, and how is the in-flight INSERT behaviour change justified in a minor release?
4. What does the fresh MVCC snapshot per lookup attempt cost the apply worker on an update-heavy workload, compared with master?
5. Does the fix leave check_exclusion_or_unique_constraint and other SnapshotDirty callers as they are, and is that intended?

## Touch points

- **WAL / replication / standby:** The fix changes how the logical replication apply worker finds the subscriber row in RelationFindReplTupleByIndex.
- **backpatchability:** The CommitFest entry targets stable, and the fix changes conflict outcomes that users of current releases can observe.
- **maintenance burden ("who maintains this in 10 years"):** The apply worker lookup gets a new snapshot rule, and new injection points sit in heapam_index_getnext_slot().
- **performance on the hot path:** The apply worker takes a fresh snapshot with GetLatestSnapshot before each lookup attempt, and no benchmark is shown.

## Who would commit it, and when

- Owner: silent. Someone who knows the logical replication apply worker and conflict detection, with a second opinion from someone who knows nbtree scans with non-MVCC snapshots. The brief shows Amit Kapila questioning the premise and suggesting documentation; no committer in the visible material has agreed to the fix.
- Priority: needs champion; audience a sizeable minority; review cost medium. The fix prevents lost publisher updates and deletes on subscribers that also take local writes, and the current design is confined to the tuple lookup of the apply worker. The entry has been in 11 CommitFests, and no committer in the visible material has ruled on the in-flight INSERT trade-off or on back-patching.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
