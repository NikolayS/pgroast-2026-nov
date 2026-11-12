# Design pre-review: Timeline switching with partial WAL records can break replica recovery

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6038](https://commitfest.postgresql.org/patch/6038/) (Needs review); authors: Alyona Vinter (dlaaren)
- Thread: [Timeline switching with partial WAL records can break replica recovery](https://www.postgresql.org/message-id/flat/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), started 2025-06-15
- Messages read: 10 of 10
- Patch prose read: v1-0001-Fix-missing-XLOG_OVERWRITE_CONTRECORD-in-timeline_switches.patch, v1-0002-Replace-assert-with-if-check-in-walsummarizer.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The author dropped the posted design (v1 to v3) in message #10, and the replacement, an overwritten_lsn field in the end-of-recovery record, has no patch. Four design points on it are unanswered: replication-ondisk-2, standards-compat-1, prior-attempts-2 and replication-ondisk-3. maintenance-fit-3 asks for a reproducer on a tree with commit 6cf1647; two field reports exist, so that point alone does not stop the review. The goal is a real bug fix, and the list must first agree on a design.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, and only 3 of 10 messages were shown in full. The overwritten_lsn design has no patch, so its effect on the WAL reader, pg_rewind and pg_waldump could not be judged.

## The proposal, restated

**Problem.** A promotion during a multi-page WAL record write leaves a partial record at the end of timeline N, and the new timeline has no XLOG_OVERWRITE_CONTRECORD marking it. A downstream replica in a cascade then keeps asking for the missing page and loops forever in XLogReadPage. A second report shows a cascading walsender failing Assert("sentPtr <= SendRqstPtr") in a related case.

**Proposed design.** Before the new timeline starts, the patch writes XLOG_OVERWRITE_CONTRECORD at the end of the old timeline to mark the partial record. It also replaces an assertion in walsummarizer with a check, and v3 makes pg_rewind start its checkpoint search from the target's last checkpoint. After review, the author now suggests leaving timeline N alone and adding an overwritten_lsn field to the end-of-recovery record.

**User-visible surface.**
- New WAL content at a timeline switch: an XLOG_OVERWRITE_CONTRECORD record at the end of the old timeline (v1 to v3).
- Proposed new field overwritten_lsn in the end-of-recovery WAL record (message #10, no patch yet).
- Changed pg_rewind behaviour when the common timeline ends with an overwritten contrecord (v3-0003).

**What the author claims.**
- The fix addresses the root cause, while commit 6cf1647d87e7cd423d71525a8759b75c4e4a47ec only treats the symptom.
- Writing the marker is safe because no other WAL writes can happen during the timeline switch.
- A second poster says the same patch also fixes the cascading walsender assertion and a later PANIC on the replica.

**Thread state.** Artem Gavrilov objected that writing into timeline N creates a second version of it that can conflict with already archived WAL. The author agreed and floated an overwritten_lsn field in the end-of-recovery record, with no patch or reply since January 2026.

## 2. Prior attempts

### [BUG] Panic due to incorrect missingContrecPtr after promotion (related)

- **Link:** [https://commitfest.postgresql.org/patch/3675/](https://commitfest.postgresql.org/patch/3675/)
- **Thread:** [https://www.postgresql.org/message-id/flat/44D259DE-7542-49C4-8A52-2AB01534DCA9@amazon.com](https://www.postgresql.org/message-id/flat/44D259DE-7542-49C4-8A52-2AB01534DCA9@amazon.com)
- **Date:** decided 2022-09-21; started 2022-02-22
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch was not committed because a separate commit, 6672d79, fixed the same issue. Robert Haas pointed to that commit, and the author confirmed it fixes the PANIC. Before that, Kyotaro Horiguchi's debugging of the author's logs suggested the inconsistent WAL came from the primary.
- **Reason (quote):** "For the benefit of anyone who may be looking at this thread in the archive later, I believe this commit will have fixed this issue:" ([message](https://www.postgresql.org/message-id/CA%2BTgmoY%2BWgqfj%2BdLdL_ESaf4YX8hKSDkkfXfZvHrsPPzZ2o_1w%40mail.gmail.com))
- **What is different this time:** P3 fixed a stale missingContrecPtr that wrote an overwrite contrecord at a past LSN after promotion. This proposal targets a partial record left at a timeline switch and a cascade replica loop.


## 3. Objections a reviewer is likely to raise

### 1. The stuck replica may never reach the record that carries overwritten_lsn

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-2`

A downstream replica has the partial record from timeline N in pg_wal and waits in XLogReadPage for its next page. The overwritten_lsn value sits in the end-of-recovery record at the start of timeline N+1, which the reader has not decoded yet. The replica keeps looping, and the operator sees no change after the upgrade.

Message #2 shows that the loop happens while the reader is still assembling the partial record, before any record of timeline N+1 is decoded. With timeline N left unchanged (message #10), the reader must know to drop a half-read record because of information found later in the stream. That changes how the WAL reader decides where valid WAL ends, a step that earlier reviewers judged risky [P4]. The proposal does not say where in xlogreader or xlogrecovery this decision is made.

**Precedents.**
- [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (Withdrawn, 2024-01-31); thread: [link](https://www.postgresql.org/message-id/flat/20200228.160100.2210969269596489579.horikyota.ntt@gmail.com)

Thread: [#2 (D Laaren, 2025-06-17)](https://www.postgresql.org/message-id/CAGWv16%2BhDSNThZeNf0qvUHHpmLE04jurrqHN7BbV1_uSN6tq%2Bw%40mail.gmail.com), [#10 (Alena Vinter, 2026-01-14)](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com)

**Question:** At which point does the replica learn about overwritten_lsn before it starts waiting for the missing page, and what keeps a real torn record from being skipped?

Counter-check: held. Message #2 shows the loop happens while the reader waits for the rest of the partial record. Message #10 does not say how the reader learns of overwritten_lsn before that wait. The objection stands without [P4].

### 2. Changing the end-of-recovery record format blocks a backpatch of this bugfix

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby · `standards-compat-1`

An operator applies a minor release on a cascading standby but not yet on the replica below it. The cascading standby is promoted and writes an end-of-recovery record with the new overwritten_lsn field. The downstream replica, still on the older minor release, misreads or rejects that record and stops following the new timeline.

The entry is tagged Bugfix, and the loop affects released branches, so users will expect a backpatch. Message #10 proposes adding overwritten_lsn to the end-of-recovery record, which changes the WAL record format. Postgres avoids WAL format changes in minor releases because standbys and primaries on different minor versions must read each other's WAL. The thread does not say which branches get the fix, or whether older branches would get a different fix.

Thread: [#10 (Alena Vinter, 2026-01-14)](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com)

**Question:** How will the overwritten_lsn design be fixed in supported branches without a WAL format change that older minor-release standbys cannot read?

Counter-check: held. The entry is tagged Bugfix and message #10 changes the end-of-recovery record. No message names the branches or says how a replica on an older minor release reads the record.

### 3. Marking the partial record in the new timeline was dropped once already

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade, backpatchability · `prior-attempts-2`

The author follows message #10 and records the overwritten LSN in the end-of-recovery record on timeline N+1. Replicas, pg_rewind and pg_waldump must then treat a page on N+1 as a valid continuation of a record that the new timeline never contains. If the zero-gap problem behind the earlier decision comes back, a replica or pg_rewind can misjudge where valid WAL ends.

Message #1 says the original contrecord work proposed writing XLOG_OVERWRITE_CONTRECORD in the new timeline, and that this was avoided because of concerns about zero-gaps in WAL. Message #10 returns to marking the incomplete record from the new timeline but does not say how that concern is answered. The thread also does not say how a new field in the end-of-recovery record could be backpatched. [P4] was withdrawn partly because reviewers judged extra end-of-WAL logic in the WAL reader too risky for the gain.

**Precedents.**
- [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (Withdrawn, 2024-01-31); thread: [link](https://www.postgresql.org/message-id/flat/20200228.160100.2210969269596489579.horikyota.ntt@gmail.com)

Thread: [#1 (D Laaren, 2025-06-15)](https://www.postgresql.org/message-id/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), [#10 (Alena Vinter, 2026-01-14)](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com)

**Question:** How does the overwritten_lsn design avoid the zero-gap problem that led to dropping the new-timeline marker, and how would it reach back branches without a WAL format change?

Counter-check: held. Message #1 records that the new-timeline marker was dropped over zero-gap concerns, and message #10 does not answer them. [P4] must not be used as support: it weighed risk against a cosmetic message change, while this is a bug fix.

### 4. New design is not shown to fix the cascading walsender case

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-3`

An upstream replica's walreceiver rewrites WAL after the primary switches timeline at the partial record. Its cascading walsender then resends data it already sent, so a debug build fails Assert("sentPtr <= SendRqstPtr") and a production build may send divergent WAL that later ends in a PANIC downstream.

Message #5 reports that v1 fixed this case because the marker was written into timeline N. Messages #8 to #10 drop that approach, and the overwritten_lsn idea does not touch what an upstream walsender has already sent. The thread does not say whether this second failure still needs a walsender change, which message #5 considered and set aside.

Thread: [#5 (Nataliia, 2025-09-04)](https://www.postgresql.org/message-id/CANbKXii0VsaEcxkd4m1Es_uJoX8D7BAZSVg1%2BwTVWhxGV8MOkA%40mail.gmail.com), [#10 (Alena Vinter, 2026-01-14)](https://www.postgresql.org/message-id/CAGWv16LtOA8BdL4s0t%3D7Wnn7wn%3D7hdzr%2B8H_un%2B-ZPBkwyn0AA%40mail.gmail.com)

**Question:** Does the overwritten_lsn design prevent the cascading walsender from resending rewritten WAL, and which test shows it?

Counter-check: held. Message #5 says the v1 approach repairs the walsender case, and messages #8 to #10 drop that approach. No message says whether the overwritten_lsn design covers the walsender assertion.

### 5. No case shown that still fails after the existing commit

fixable · level 1b, rubric item 2 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A committer takes on a change to timeline switch logic and to the WAL record format. Without a scenario that fails on current master, the committer cannot tell whether the change fixes a remaining bug or repeats commit 6cf1647. The committer would then own the risk of the change with no measured gain.

Message #2 calls commit 6cf1647 a fix for the symptom but does not give a scenario that still fails with it applied. Message #1 says its TAP test does not trigger the loop reliably. The reproducer in message #5 deletes a WAL file by hand. Message #5 does report a real assertion failure, but it does not say whether that report was made on a branch that has 6cf1647.

Thread: [#1 (D Laaren, 2025-06-15)](https://www.postgresql.org/message-id/CAGWv16KJg%3DSHKxwTWxGFo9zibmEompk%3DbfK0MRHTEh8NipoyHQ%40mail.gmail.com), [#2 (D Laaren, 2025-06-17)](https://www.postgresql.org/message-id/CAGWv16%2BhDSNThZeNf0qvUHHpmLE04jurrqHN7BbV1_uSN6tq%2Bw%40mail.gmail.com), [#5 (Nataliia, 2025-09-04)](https://www.postgresql.org/message-id/CANbKXii0VsaEcxkd4m1Es_uJoX8D7BAZSVg1%2BwTVWhxGV8MOkA%40mail.gmail.com)

**Question:** Which deterministic scenario still loops or hits the walsender assertion on current master, which includes commit 6cf1647?

Counter-check: held. Message #2 names commit 6cf1647 as the current fix, but no message shows a failure on a tree that has it. The test in message #1 is unreliable, and the test in message #5 removes a WAL file by hand.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection prior-attempts-2: Marking the partial record in the new timeline was dropped once already |
| WAL / replication / standby | **yes** | The proposal changes what promotion writes to WAL and how replicas, walsummarizer and pg_rewind handle a timeline switch. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | A new field in the end-of-recovery record changes a layout that external WAL readers decode (message #7). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, and the current design changes a WAL record format that minor releases normally keep stable. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Recovery, walsummarizer, pg_rewind and pg_waldump would each need to handle a second way to mark an overwritten contrecord. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection maintenance-fit-3: No case shown that still fails after the existing commit |

## 5. Questions the author must answer before anyone reviews code

1. Which deterministic scenario still loops or hits the walsender assertion on current master, which includes commit 6cf1647?
2. At which point does the replica learn about overwritten_lsn before it waits for the missing page, and what keeps a real torn record from being skipped?
3. Which supported branches get a fix, and how does a standby on an older minor release read an end-of-recovery record with the new field?
4. How does the overwritten_lsn design answer the zero-gap concern that led to dropping the marker in the new timeline?
5. Does the overwritten_lsn design stop the cascading walsender from resending rewritten WAL, and which test shows it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the WAL reader, the end-of-recovery and timeline switch code, and pg_rewind. No committer has posted in the thread.
- **Liability:** The owner signs for WAL written at every promotion and for how each replica decides where a timeline ends. A mistake can stall or corrupt standbys, and a changed record format stays for the life of each branch.
- **Author follow-up:** The author posted three versions and accepted the review in message #10. No patch or reply has followed since January 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The failure needs a cascading setup and a promotion in the middle of a multi-page record, but it leaves a replica stuck. The posted patches follow a design the author dropped, and the replacement touches recovery, pg_rewind and the WAL record format with no patch yet.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Two different ways to mark an overwritten contrecord. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-1` No evidence the loop survives the existing fix in 6cf1647d. Refuted (not specific): Duplicate of maintenance-fit-3. Its support from [P3] also does not hold: [P3] concerned a different PANIC fixed by commit 6672d79, which is a different commit from 6cf1647d.
- `replication-ondisk-1` Overwritten_lsn in the end-of-recovery record changes WAL format for a bugfix. Refuted (not specific): Duplicate of standards-compat-1. The third-party tool point is covered by standards-compat-2.
- `maintenance-fit-1` Proposed bug fix needs a WAL format change it cannot backpatch. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-2` Each WAL consumer must learn the new marker, and failures stay silent. Refuted (not specific): Duplicate of standards-compat-2 for the point that every WAL reader must learn the new field. Its own failure scenario is an unspecified future change, and [P4] concerned a cosmetic message change.
- `already-exists-1` Proposal does not show the loop survives the existing fix in 6cf1647d87e7. Refuted (not specific): Duplicate of maintenance-fit-3. [P3] does not support it either, because that entry was closed by commit 6672d79 for a different PANIC.
- `demand-1` No evidence the loop still happens on master after the existing fix. Refuted (not specific): Duplicate of maintenance-fit-3.

### Review angles

- prior-attempts: 2 objection(s). The v1-v3 design of writing into timeline N was settled in messages #7-#10, where the author agreed it conflicts with archived WAL, so it is not raised here. The thread itself cites two earlier discussions (message #1) and commit 6cf1647d (message #2).
- standards-compat: 2 objection(s). The concern about writing into an already archived timeline N was raised in message #7 and accepted by the author in message #10, so it is not repeated here.
- replication-ondisk: 3 objection(s). The v1-v3 design of writing into timeline N is not raised again here, because message #7 objected to it and message #8 accepted that.
- security: 0 objection(s). The proposal changes WAL written at promotion, the replica's handling of a timeline switch, and pg_rewind's checkpoint search, and only superusers or operators start these actions. No new privilege, role, setting, information exposure, or path that lets an unprivileged user cause a denial of service is involved.
- maintenance-fit: 3 objection(s). No committer has posted in the thread in the ten months it has been open. Only a non-committer reviewer is engaged, so ownership of the recovery and pg_rewind changes is open.
- already-exists: 1 objection(s). Only one point fits this angle. No extension or external tool can fix how the server writes WAL at a timeline switch, so core placement is not in question.
- demand: 1 objection(s). This is a bug fix, so the usual demand test does not apply. Two posters report two field failures in cascading setups, and a reviewer is engaged.

### Precedents retrieved

- P1: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P2: [using an end-of-recovery record in all cases](https://commitfest.postgresql.org/patch/3365/) (cf, Withdrawn, 2022-01-17) - judged not relevant; reason summarised from its thread
- P3: [\[BUG\] Panic due to incorrect missingContrecPtr after promotion](https://commitfest.postgresql.org/patch/3675/) (cf, Withdrawn, 2022-09-21) - used in the review; reason summarised from its thread
- P4: [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (cf, Withdrawn, 2024-01-31) - used in the review; reason summarised from its thread
- P5: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P6: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - used in the review; reason summarised from its thread
- P7: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P8: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant; reason summarised from its thread
- P9: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P10: [Add DB_IN_END_OF_RECOVERY_CHECKPOINT state for control file](https://commitfest.postgresql.org/patch/3442/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P11: [get current and recovery timeline](https://commitfest.postgresql.org/patch/2225/) (cf, Returned with feedback, 2020-01-31) - judged not relevant
- P12: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P13: [Logical decoding timeline following take II](https://commitfest.postgresql.org/patch/779/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P14: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P15: [Revert the patch to check if we've reached end-of-backup also when doing crash recovery, and throw an error if not.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54685b1c2b356b4b9c0938f6b8dcc52e173c0400) (revert, Reverted, 2011-04-13) - judged not relevant
- P16: [Revert "Disallow infinite endpoints in generate_series() for timestamps."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29904f5f2fdafbbb96ef3685fd361053b061aeb1) (revert, Reverted, 2022-05-09) - judged not relevant
- P17: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P18: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P19: [OOM in libpq and infinite loop with getCopyStart()](https://commitfest.postgresql.org/patch/565/) (cf, Returned with feedback, 2017-01-23) - judged not relevant
- P20: [64-bit TOAST value ID](https://commitfest.postgresql.org/patch/4296/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P21: [DELETE CASCADE](https://commitfest.postgresql.org/patch/3195/) (cf, Returned with feedback, 2022-01-31) - judged not relevant
- P22: [Make pg_rewind not copy useless WALs](https://commitfest.postgresql.org/patch/1302/) (cf, Returned with feedback, 2018-01-31) - judged not relevant
- P23: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P24: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 186 s; cost reported by the CLI: $2.59
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 107 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:38:55.713Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
