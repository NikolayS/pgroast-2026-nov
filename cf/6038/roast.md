# Design pre-review: Timeline switching with partial WAL records can break replica recovery

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6038](https://commitfest.postgresql.org/patch/6038/); [thread](https://www.postgresql.org/message-id/flat/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The author dropped the posted design (v1 to v3) in message #10, and the replacement, an overwritten_lsn field in the end-of-recovery record, has no patch. Four design points on it are unanswered: replication-ondisk-2, standards-compat-1, prior-attempts-2 and replication-ondisk-3. maintenance-fit-3 asks for a reproducer on a tree with commit 6cf1647; two field reports exist, so that point alone does not stop the review. The goal is a real bug fix, and the list must first agree on a design.

Not checked: No code was read, and only 3 of 10 messages were shown in full. The overwritten_lsn design has no patch, so its effect on the WAL reader, pg_rewind and pg_waldump could not be judged.

## Prior attempts

- [\[BUG\] Panic due to incorrect missingContrecPtr after promotion](https://www.postgresql.org/message-id/flat/44D259DE-7542-49C4-8A52-2AB01534DCA9@amazon.com) (related; Withdrawn, 2022-09-21). "For the benefit of anyone who may be looking at this thread in the archive later, I believe this commit will have fixed this issue:" ([message](https://www.postgresql.org/message-id/CA%2BTgmoY%2BWgqfj%2BdLdL_ESaf4YX8hKSDkkfXfZvHrsPPzZ2o_1w%40mail.gmail.com)) Different now: P3 fixed a stale missingContrecPtr that wrote an overwrite contrecord at a past LSN after promotion. This proposal targets a partial record left at a timeline switch and a cascade replica loop.

## Objections a reviewer is likely to raise

### 1. The stuck replica may never reach the record that carries overwritten_lsn

A downstream replica has the partial record from timeline N in pg_wal and waits in XLogReadPage for its next page. The overwritten_lsn value sits in the end-of-recovery record at the start of timeline N+1, which the reader has not decoded yet. The replica keeps looping, and the operator sees no change after the upgrade.

Message #2 shows that the loop happens while the reader is still assembling the partial record, before any record of timeline N+1 is decoded. With timeline N left unchanged (message #10), the reader must know to drop a half-read record because of information found later in the stream. That changes how the WAL reader decides where valid WAL ends, a step that earlier reviewers judged risky [P4]. The proposal does not say where in xlogreader or xlogrecovery this decision is made.

Precedent: [Make message at end-of-recovery less scary](https://www.postgresql.org/message-id/flat/20200228.160100.2210969269596489579.horikyota.ntt@gmail.com) (Withdrawn, 2024-01-31)

**Question:** At which point does the replica learn about overwritten_lsn before it starts waiting for the missing page, and what keeps a real torn record from being skipped?

<sub>fixable, level 2, WAL / replication / standby; messages [#2](https://www.postgresql.org/message-id/CAGWv16%2BhDSNThZeNf0qvUHHpmLE04jurrqHN7BbV1_uSN6tq%2Bw%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com); `replication-ondisk-2`</sub>

### 2. Changing the end-of-recovery record format blocks a backpatch of this bugfix

An operator applies a minor release on a cascading standby but not yet on the replica below it. The cascading standby is promoted and writes an end-of-recovery record with the new overwritten_lsn field. The downstream replica, still on the older minor release, misreads or rejects that record and stops following the new timeline.

The entry is tagged Bugfix, and the loop affects released branches, so users will expect a backpatch. Message #10 proposes adding overwritten_lsn to the end-of-recovery record, which changes the WAL record format. Postgres avoids WAL format changes in minor releases because standbys and primaries on different minor versions must read each other's WAL. The thread does not say which branches get the fix, or whether older branches would get a different fix.

**Question:** How will the overwritten_lsn design be fixed in supported branches without a WAL format change that older minor-release standbys cannot read?

<sub>fixable, level 2, backpatchability; messages [#10](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Marking the partial record in the new timeline was dropped once already

The author follows message #10 and records the overwritten LSN in the end-of-recovery record on timeline N+1. Replicas, pg_rewind and pg_waldump must then treat a page on N+1 as a valid continuation of a record that the new timeline never contains. If the zero-gap problem behind the earlier decision comes back, a replica or pg_rewind can misjudge where valid WAL ends.

Message #1 says the original contrecord work proposed writing XLOG_OVERWRITE_CONTRECORD in the new timeline, and that this was avoided because of concerns about zero-gaps in WAL. Message #10 returns to marking the incomplete record from the new timeline but does not say how that concern is answered. The thread also does not say how a new field in the end-of-recovery record could be backpatched. [P4] was withdrawn partly because reviewers judged extra end-of-WAL logic in the WAL reader too risky for the gain.

Precedent: [Make message at end-of-recovery less scary](https://www.postgresql.org/message-id/flat/20200228.160100.2210969269596489579.horikyota.ntt@gmail.com) (Withdrawn, 2024-01-31)

**Question:** How does the overwritten_lsn design avoid the zero-gap problem that led to dropping the new-timeline marker, and how would it reach back branches without a WAL format change?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com); `prior-attempts-2`</sub>

### 4. New design is not shown to fix the cascading walsender case

An upstream replica's walreceiver rewrites WAL after the primary switches timeline at the partial record. Its cascading walsender then resends data it already sent, so a debug build fails Assert("sentPtr <= SendRqstPtr") and a production build may send divergent WAL that later ends in a PANIC downstream.

Message #5 reports that v1 fixed this case because the marker was written into timeline N. Messages #8 to #10 drop that approach, and the overwritten_lsn idea does not touch what an upstream walsender has already sent. The thread does not say whether this second failure still needs a walsender change, which message #5 considered and set aside.

**Question:** Does the overwritten_lsn design prevent the cascading walsender from resending rewritten WAL, and which test shows it?

<sub>fixable, level 2, WAL / replication / standby; messages [#5](https://www.postgresql.org/message-id/CANbKXii0VsaEcxkd4m1Es_uJoX8D7BAZSVg1%2BwTVWhxGV8MOkA%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com); `replication-ondisk-3`</sub>

### 5. No case shown that still fails after the existing commit

A committer takes on a change to timeline switch logic and to the WAL record format. Without a scenario that fails on current master, the committer cannot tell whether the change fixes a remaining bug or repeats commit 6cf1647. The committer would then own the risk of the change with no measured gain.

Message #2 calls commit 6cf1647 a fix for the symptom but does not give a scenario that still fails with it applied. Message #1 says its TAP test does not trigger the loop reliably. The reproducer in message #5 deletes a WAL file by hand. Message #5 does report a real assertion failure, but it does not say whether that report was made on a branch that has 6cf1647.

**Question:** Which deterministic scenario still loops or hits the walsender assertion on current master, which includes commit 6cf1647?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAGWv16%2BhDSNThZeNf0qvUHHpmLE04jurrqHN7BbV1_uSN6tq%2Bw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CANbKXii0VsaEcxkd4m1Es_uJoX8D7BAZSVg1%2BwTVWhxGV8MOkA%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Which deterministic scenario still loops or hits the walsender assertion on current master, which includes commit 6cf1647?
2. At which point does the replica learn about overwritten_lsn before it waits for the missing page, and what keeps a real torn record from being skipped?
3. Which supported branches get a fix, and how does a standby on an older minor release read an end-of-recovery record with the new field?
4. How does the overwritten_lsn design answer the zero-gap concern that led to dropping the marker in the new timeline?
5. Does the overwritten_lsn design stop the cascading walsender from resending rewritten WAL, and which test shows it?

## Touch points

- **on-disk format / pg_upgrade:** see objection prior-attempts-2: Marking the partial record in the new timeline was dropped once already
- **WAL / replication / standby:** The proposal changes what promotion writes to WAL and how replicas, walsummarizer and pg_rewind handle a timeline switch.
- **ABI / extension API:** A new field in the end-of-recovery record changes a layout that external WAL readers decode (message #7).
- **backpatchability:** The entry is tagged Bugfix, and the current design changes a WAL record format that minor releases normally keep stable.
- **maintenance burden ("who maintains this in 10 years"):** Recovery, walsummarizer, pg_rewind and pg_waldump would each need to handle a second way to mark an overwritten contrecord.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection maintenance-fit-3: No case shown that still fails after the existing commit

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the WAL reader, the end-of-recovery and timeline switch code, and pg_rewind. No committer has posted in the thread.
- Priority: needs design rethink; audience niche; review cost high. The failure needs a cascading setup and a promotion in the middle of a multi-page record, but it leaves a replica stuck. The posted patches follow a design the author dropped, and the replacement touches recovery, pg_rewind and the WAL record format with no patch yet.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
