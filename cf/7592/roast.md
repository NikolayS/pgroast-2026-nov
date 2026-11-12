# Design pre-review: Assertion failure in WAL summarizer after promotion

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7592](https://commitfest.postgresql.org/patch/7592/); [thread](https://www.postgresql.org/message-id/flat/CAHGQGwFbpzR2Wkrku9J8w6E%2B-rzD6Ovazs8n_kxvzfccfnJ0tg%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the design. The series fixes a reproduced assertion failure and a silent wrong restore from incremental backup. The two surviving objections are fixable: maintenance-fit-1 asks for a branch plan for 0002 and 0003, and replication-ondisk-1 asks how a user recovers after 0002 refuses a backup. Both are decisions for a committer who knows the summarizer and backpatch policy.

Not checked: Only the v1-0001 prose and 3 of 13 messages were read. The 0002 check, its error text, and whether 0003 changes other callers of GetLatestLSN() could not be judged without the code.

## Prior attempts

None found before 2026-10-22.

## Objections a reviewer is likely to raise

### 1. Backpatch scope for the data-loss fixes 0002 and 0003 is not stated

A committer pushes 0001 with its Backpatch-through: 17 tag and leaves 0002 and 0003 on master only. A user on 17 promotes a standby that was paused at a recovery target and later takes an incremental backup. That backup restores old table data without any error, and the user finds out only when the restored data is wrong.

Only v1-0001 carries a Backpatch-through line. Message #2 says 0001 applies to 17-19, but messages #6 and #8 put the silent data loss in 0002 and 0003, and they say nothing about branches. If 0003 is backpatched, standbys on stable branches change behavior, because pg_available_wal_summaries() shows less WAL there. A committer would likely ask for the branch plan for each patch before taking ownership of all three.

**Question:** Which of 0002 and 0003 are meant for 17, 18 and 19, and what does a user on those branches see when 0002 refuses a backup?

<sub>fixable, level 2, backpatchability; messages [#2](https://www.postgresql.org/message-id/179263656196.983929.15753900762595867225%40gmail.com), [#6](https://www.postgresql.org/message-id/179296590548.3326877.13182798945037392623%40gmail.com), [#8](https://www.postgresql.org/message-id/179371255481.1322117.11879017402330770955%40gmail.com), [#9](https://www.postgresql.org/message-id/CA%2BTgmoYpRDgP4sh6FxKD3sCAbQ8vnyeEhd3doAM%2B-ggN7RAXfQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. 0002 refusal gives users no stated way back to working incremental backups

A user promoted a standby on 17 and the old timeline summary on disk crosses the switch point and records a truncation. After a minor update with 0002, every scheduled incremental backup fails. The proposal does not say whether the user must take a new full backup, delete summary files, or wait.

0002 exists because bad summaries already on disk cannot be repaired: entries carry no LSNs (message #6). That on-disk state persists after the fix, so the refusal can repeat on every backup. The material shows no error text, hint, or documented remedy, and it does not say whether 0002 is backpatched with 0001. A committer would likely ask for both before shipping a new hard failure in back branches.

**Question:** What error and hint does a user see when 0002 refuses, does a new full backup clear the condition, and is 0002 meant for back branches?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#6](https://www.postgresql.org/message-id/179296590548.3326877.13182798945037392623%40gmail.com), [#8](https://www.postgresql.org/message-id/179371255481.1322117.11879017402330770955%40gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which of 0002 and 0003 are meant for 17, 18 and 19, and which only for master?
2. What error message and hint does a user see when 0002 refuses an incremental backup?
3. Does a new full backup clear the 0002 refusal, or must the user also remove old summary files?
4. Does stopping summarization at the replay position delay or block incremental backups taken from a lagging standby?
5. Which commit does v3 build on, and do the back branches need it before 0002 and 0003 apply?

## Touch points

- **on-disk format / pg_upgrade:** 0002 refuses WAL summary files already on disk that record a truncation and cross a timeline switch point.
- **WAL / replication / standby:** 0003 makes the standby WAL summarizer stop at the replay position, and 0001 changes its behavior after promotion.
- **backpatchability:** 0001 is tagged Backpatch-through: 17, and the thread gives no branch plan for 0002 and 0003.
- **maintenance burden ("who maintains this in 10 years"):** 0002 adds a refusal path in incremental backup that must stay while old summaries can exist.

## Who would commit it, and when

- Owner: qualified committer engaged. Someone who knows the WAL summarizer and incremental backup code must sign. Fujii Masao reported the bug and Robert Haas answered the 0003 question, but neither has said whether the series goes in.
- Priority: proceed; audience a sizeable minority; review cost medium. The series fixes a silent wrong restore for users of summarize_wal and incremental backup who promote a standby, and it adds no new user-facing setting or format. Review cost is medium because no code was read and the three patches touch the summarizer and incremental backup.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
