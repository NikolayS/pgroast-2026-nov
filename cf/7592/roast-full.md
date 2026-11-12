# Design pre-review: Assertion failure in WAL summarizer after promotion

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7592](https://commitfest.postgresql.org/patch/7592/) (Needs review); authors: shihao zhong (zhong950419), Manuel Reyes Bravo (manu15)
- Thread: [Assertion failure in WAL summarizer](https://www.postgresql.org/message-id/flat/CAHGQGwFbpzR2Wkrku9J8w6E%2B-rzD6Ovazs8n_kxvzfccfnJ0tg%40mail.gmail.com), started 2026-10-22
- Messages read: 13 of 13
- Patch prose read: v1-0001-Fix-assertion-failure-in-WAL-summarizer-after-pro.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the design. The series fixes a reproduced assertion failure and a silent wrong restore from incremental backup. The two surviving objections are fixable: maintenance-fit-1 asks for a branch plan for 0002 and 0003, and replication-ondisk-1 asks how a user recovers after 0002 refuses a backup. Both are decisions for a committer who knows the summarizer and backpatch policy.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the v1-0001 prose and 3 of 13 messages were read. The 0002 check, its error text, and whether 0003 changes other callers of GetLatestLSN() could not be judged without the code.

## The proposal, restated

**Problem.** After a standby paused at a recovery target is promoted, the WAL summarizer, which has summarized past the target, fails Assert("end_of_summary_lsn >= current_lsn") and the postmaster restarts the cluster. The same root cause lets a timeline 1 summary cover WAL past the switch point, so an incremental backup on the promoted server can skip changed blocks and restore old data without error. This affects users of summarize_wal and incremental backup since 17.

**Proposed design.** The v4 series has three patches. 0001 makes SummarizeWAL() report no progress when read_upto moved back before the summary start, so the main loop rewinds to the switch point. 0002 makes incremental backup refuse a truncation recorded in a summary that crosses a timeline switch point, and 0003 makes GetLatestLSN() return the replay position during recovery, so standbys never summarize past replay.

**User-visible surface.**
- An incremental backup now fails when an existing WAL summary records a truncation and crosses a timeline switch point (0002).
- On a standby the WAL summarizer now stops at the replay position instead of the walreceiver flush position, so pg_available_wal_summaries() shows less WAL there (0003).

**What the author claims.**
- The assertion failure goes back to 17 and v1/v2 of the fix apply to master, 19, 18 and 17.
- Builds without assertions recover from the assertion case on their own, with the same summaries and state as with 0001.
- Shihao's test restores a table with 4|0 instead of 4|1 on master without cassert (3 of 3 runs) and passes with the series.
- 0003 alone does not fix summaries already on disk, so 0002 refuses them, because summary entries carry no LSNs.
- Robert Haas states that nothing relies on standby summaries ahead of replay.

**Thread state.** Fujii reported the bug; Daria and Sai Pranav confirmed it and tested, Robert answered the 0003 question, and Sai Pranav gave LGTM on v4. Open: 0003 commit message names stale test file names, and no committer has said whether the series goes in or how far it is backpatched.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-22; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatch scope for the data-loss fixes 0002 and 0003 is not stated

fixable · level 2, rubric item 6 · backpatchability, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A committer pushes 0001 with its Backpatch-through: 17 tag and leaves 0002 and 0003 on master only. A user on 17 promotes a standby that was paused at a recovery target and later takes an incremental backup. That backup restores old table data without any error, and the user finds out only when the restored data is wrong.

Only v1-0001 carries a Backpatch-through line. Message #2 says 0001 applies to 17-19, but messages #6 and #8 put the silent data loss in 0002 and 0003, and they say nothing about branches. If 0003 is backpatched, standbys on stable branches change behavior, because pg_available_wal_summaries() shows less WAL there. A committer would likely ask for the branch plan for each patch before taking ownership of all three.

Thread: [#2 (Manu, 2026-10-22)](https://www.postgresql.org/message-id/179263656196.983929.15753900762595867225%40gmail.com), [#6 (Manu, 2026-10-25)](https://www.postgresql.org/message-id/179296590548.3326877.13182798945037392623%40gmail.com), [#8 (Manu, 2026-11-03)](https://www.postgresql.org/message-id/179371255481.1322117.11879017402330770955%40gmail.com), [#9 (Robert Haas, 2026-11-03)](https://www.postgresql.org/message-id/CA%2BTgmoYpRDgP4sh6FxKD3sCAbQ8vnyeEhd3doAM%2B-ggN7RAXfQ%40mail.gmail.com)

**Question:** Which of 0002 and 0003 are meant for 17, 18 and 19, and what does a user on those branches see when 0002 refuses a backup?

Counter-check: held. Only 0001 carries a Backpatch-through line, while messages #5 and #6 tie the silent data loss to the 0002 and 0003 fixes. No message gives a branch plan for those two, and 0003 changes standby behavior if backpatched.

### 2. 0002 refusal gives users no stated way back to working incremental backups

fixable · level 2, rubric item 14 · on-disk format / pg_upgrade, backpatchability · `replication-ondisk-1`

A user promoted a standby on 17 and the old timeline summary on disk crosses the switch point and records a truncation. After a minor update with 0002, every scheduled incremental backup fails. The proposal does not say whether the user must take a new full backup, delete summary files, or wait.

0002 exists because bad summaries already on disk cannot be repaired: entries carry no LSNs (message #6). That on-disk state persists after the fix, so the refusal can repeat on every backup. The material shows no error text, hint, or documented remedy, and it does not say whether 0002 is backpatched with 0001. A committer would likely ask for both before shipping a new hard failure in back branches.

Thread: [#6 (Manu, 2026-10-25)](https://www.postgresql.org/message-id/179296590548.3326877.13182798945037392623%40gmail.com), [#8 (Manu, 2026-11-03)](https://www.postgresql.org/message-id/179371255481.1322117.11879017402330770955%40gmail.com)

**Question:** What error and hint does a user see when 0002 refuses, does a new full backup clear the condition, and is 0002 meant for back branches?

Counter-check: held. Message #6 confirms 0002 refuses summaries already on disk, and the thread does not say how a user gets back to working incremental backups. A new full backup probably clears the condition, but the proposal does not say so.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | 0002 refuses WAL summary files already on disk that record a truncation and cross a timeline switch point. |
| WAL / replication / standby | **yes** | 0003 makes the standby WAL summarizer stop at the replay position, and 0001 changes its behavior after promotion. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | 0001 is tagged Backpatch-through: 17, and the thread gives no branch plan for 0002 and 0003. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0002 adds a refusal path in incremental backup that must stay while old summaries can exist. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which of 0002 and 0003 are meant for 17, 18 and 19, and which only for master?
2. What error message and hint does a user see when 0002 refuses an incremental backup?
3. Does a new full backup clear the 0002 refusal, or must the user also remove old summary files?
4. Does stopping summarization at the replay position delay or block incremental backups taken from a lagging standby?
5. Which commit does v3 build on, and do the back branches need it before 0002 and 0003 apply?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Someone who knows the WAL summarizer and incremental backup code must sign. Fujii Masao reported the bug and Robert Haas answered the 0003 question, but neither has said whether the series goes in.
- **Liability:** The owner signs for incremental backup correctness across timeline switches in every branch that gets the fix. If 0002 and 0003 are backpatched, the owner also signs for a new backup refusal and changed standby summarization in minor releases.
- **Author follow-up:** The author posted four versions in about three weeks and answered review comments, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The series fixes a silent wrong restore for users of summarize_wal and incremental backup who promote a standby, and it adds no new user-facing setting or format. Review cost is medium because no code was read and the three patches touch the summarizer and incremental backup.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Refused incremental backup has no stated message or recovery path. Refuted (not specific): Duplicate of replication-ondisk-1, which states the recovery question more precisely. Its claim that the patch prose shows no error text is weak, because the material includes prose only for v1-0001 and none for 0002.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier attempt at the same or a similar idea. Inside the thread, message #6 links Ayush's and Sai Pranav's neighbouring fixes and Robert's original thread, but none of them is reported as a rejected attempt, and message #9 answers the 0003 question.
- standards-compat: 1 objection(s). 0003 changes what pg_available_wal_summaries() shows on a standby. Robert Haas says nothing relies on summaries ahead of replay (message #9), so I raise no compatibility objection about it. The patch adds no syntax, GUC or dumpable object.
- replication-ondisk: 1 objection(s). The series adds no WAL record, catalog, or protocol change. 0003 narrows standby summarization to replay, and Robert Haas confirmed in message #9 that nothing relies on summaries ahead of replay.
- security: 0 objection(s). The series has no privilege or exposure aspect. It changes internal WAL summarizer behaviour and an incremental backup check, adds no new privileges, roles, functions or settings, and does not change who can call pg_available_wal_summaries() or take backups.
- maintenance-fit: 1 objection(s). The fit is sound otherwise. This is a correctness fix in core code, with no new GUC and no hot-path cost. Robert Haas, who knows this area, said in message #9 that nothing relies on standby summaries ahead of replay.
- already-exists: 0 objection(s). This is a bug fix for an assertion failure and for silent data loss in incremental backup after promotion. No existing feature, setting or extension avoids it. The fix changes the existing summarizer and incremental backup code and adds no second mechanism, so the already-implemented angle has nothing to object to.
- demand: 0 objection(s). This is a bug fix, not a feature, so the demand test does not apply in the usual way. The thread shows a cluster restart on cassert builds and silent wrong data from incremental backup on production builds since 17. Several people reproduced it independently, in messages #2, #3, #5, #7 and #12, and Robert Haas confirmed in message #9 that nothing relies on the old standby behaviour. The user group is everyone who uses summarize_wal and incremental backup with promotion, and no workaround is shown.

### Precedents retrieved

- P1: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant; reason summarised from its thread
- P2: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Revert change to ALTER TABLESPACE summary.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=155c0f24b7d370eb57fb8dde0447e11e41c405c2) (revert, Reverted, 2014-11-14) - judged not relevant
- P4: [Revert pg_wal_replay_wait() stored procedure](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3a7ae6b3d91e0d011dba1eb8a29e1836c6a33c75) (revert, Reverted, 2024-11-04) - judged not relevant
- P5: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P6: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P7: [File level incremental backup](https://commitfest.postgresql.org/patch/94/) (cf, Rejected, 2015-03-18) - judged not relevant; reason summarised from its thread
- P8: [\[BUG\] Panic due to incorrect missingContrecPtr after promotion](https://commitfest.postgresql.org/patch/3675/) (cf, Withdrawn, 2022-09-21) - judged not relevant; reason summarised from its thread
- P9: [Be strict when request to flush past end of WAL in WaitXLogInsertionsToFinish](https://commitfest.postgresql.org/patch/7294/) (cf, Withdrawn, 2026-09-12) - judged not relevant; reason summarised from its thread
- P10: [Incremental sort](https://commitfest.postgresql.org/patch/1011/) (cf, Returned with feedback, 2017-04-03) - judged not relevant
- P11: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant; reason summarised from its thread
- P12: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant
- P13: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P14: [Revert "Temporary patch to help debug pg_walsummary test failures."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2b520860c08ac946ec5e570065ede1c467e45618) (revert, Reverted, 2024-03-20) - judged not relevant
- P15: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P16: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P17: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P18: [Track wal_level drop for backup management](https://commitfest.postgresql.org/patch/2994/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P19: [Revert "Don't truncate database and user names in startup packets."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a0ff56e2d3ff1db3de727b33b2dac985ccc43ef8) (revert, Reverted, 2024-12-12) - judged not relevant
- P20: [SimpleLruTruncate() mutual exclusion (data loss from lack thereof)](https://commitfest.postgresql.org/patch/2025/) (cf, Withdrawn, 2020-01-25) - judged not relevant
- P21: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P22: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P23: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P24: [Revert "Add basic TAP tests for the low-level backup method"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cb1b632b354518a595f1b23bfbac28a798f7f1c) (revert, Reverted, 2024-03-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 140 s; cost reported by the CLI: $1.92
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:32.114Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
