# Design pre-review: Fix timeline history after recovery stops on an ancestor

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7207](https://commitfest.postgresql.org/patch/7207/) (Needs review); authors: Stepan Filippov (etien)
- Thread: [\[PATCH\] Fix timeline history after recovery stops on an ancestor](https://www.postgresql.org/message-id/flat/29641785480901%40mail.yandex.ru), started 2026-07-31
- Messages read: 6 of 6
- Patch prose read: 0001-v2-Fix-history-after-recovery-stops-on-an-ancestor.patch, 0001-A-recovery-target-can-be-reached-before-recovery-swi.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible scenario in message #1, and the new parent matches the PrevTimeLineID that the end-of-recovery record already uses. Three reviewers agree on lastRecTLI as the parent. The open points are the back-patch decision and the empty v2 commit message, and both belong to the committer.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it is not confirmed that lastRecTLI is always valid when writeTimeLineHistory() runs, or that no other path reads the old parent. The two linked discussions in message #1 were also not available.

## The proposal, restated

**Problem.** When recovery with recovery_target_timeline = 'latest' stops at a target on timeline 1 before timeline 2 forked, Postgres writes timeline 2 as the parent of the new timeline. The resulting history file lists switchpoints that go backwards, so tools that walk timeline history, such as a WAL verifier, cannot rebuild the real ancestry.

**Proposed design.** writeTimeLineHistory() takes the timeline of the last replayed WAL record (lastRecTLI) as the parent instead of recoveryTargetTLI. The new timeline ID is still chosen from the newest timeline in the archive. 028_pitr_timelines.pl now checks the history file content, and 002_archiving.pl replays WAL on timeline 2 and waits for it before promotion.

**User-visible surface.**
- Content of the timeline history file written at end of archive recovery: the parent timeline and switchpoint change when recovery stops on an ancestor timeline.

**What the author claims.**
- The current history file does not describe the timeline switches that took place.
- The bug is a correctness issue and the fix should be back-patched.
- The new parent matches the PrevTimeLineID that the end-of-recovery record already uses.

**Thread state.** Andrey Rachitskiy, Andrey Borodin and João Rito agree that lastRecTLI is the right parent; v2 fixed a race in 002_archiving.pl that Borodin found. Rito tested master and back branches and would mark it Ready for Committer once the commit message is rewritten; Heikki and Michael were asked to comment but have not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The parent timeline and switchpoint written to a new timeline history file change; the file format stays the same. |
| WAL / replication / standby | **yes** | The change is in the end-of-archive-recovery path in xlog.c and affects the history file that standbys and archive tools read. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a back-patch, which changes history file content in minor releases without a demonstrated broken consumer. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which concrete tool or server path (pg_rewind, standby timeline choice, an archive tool) misbehaves today with the wrong history file, to justify a back-patch?
2. Does any code path or external tool depend on the old parent value, or on promotion restoring the target timeline's history file from the archive?
3. What should users do with clusters and archives that already hold a history file with backward switchpoints, and should the release notes say so?
4. Is there a case where recovery ends with lastRecTLI different from recoveryTargetTLI and recoveryTargetTLI is still the correct parent?
5. Can the v2 commit message be rewritten to explain the scenario, the choice of lastRecTLI over endOfLogTLI, and the test changes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the end-of-recovery and timeline switch code in xlog.c; message #2 asked Heikki and Michael to comment, and neither has replied in the visible thread.
- **Liability:** The committer signs for the content of history files written at end of archive recovery in every branch that receives the fix, including any effect on tools that read the old content. History files stay in archives for a long time.
- **Author follow-up:** The author posted v2 with a fix for the race in 002_archiving.pl that message #3 reported, which shows a response to review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix is a 3.8 KB patch to one call in xlog.c plus two tests, and it affects only PITR that stops on an ancestor timeline. Reviewers agree on the design, but no committer who knows the area has replied.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P11] concerns pg_rewind and diverging timelines, a different idea, and records no reason that could apply here. Messages #1 and #2 link two related discussions, but the material does not say what they concluded, so nothing from them is carried over.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is nothing to reject: no new syntax, GUC, catalog object, dump content, psql surface or protocol change. The only user-visible change is the parent timeline and switchpoint in a history file written once at end of archive recovery, and only in a case where the current content is self-contradictory. Every other consumer, such as the end-of-recovery record's PrevTimeLineID, already uses lastRecTLI (message #2, message #6). One effect is visible in the tests: a promoted standby that never replayed timeline 2 no longer restores 00000002.history from the archive (message #6, comment 3). That narrows archive reads and breaks no documented contract. The thread shows no tool that depends on the old parent value, and inventing one would break the rubric's no-invented-facts guard. Open points remain for the committer, not for rejection: back-patching a change to a minor release's history file content without a demonstrated failure (message #6, comment 4), and a release-note line for clusters that already carry such a file. [P11] concerns pg_rewind and diverging timelines, has no recorded reason, and does not support an objection here.
- replication-ondisk: 0 objection(s). The only persistent state touched is the parent and switchpoint lines that a new timeline history file records; the file format, WAL records, control file and catalogs stay the same. The new parent matches the PrevTimeLineID already written to the end-of-recovery record, so on-disk state becomes consistent. Existing wrong history files are left as they are and are not made worse. The new timeline ID is still chosen from the archive, so later promotions cannot reuse an ID. A back-patch changes no format that mixed-version standbys read. [P11] concerns pg_rewind and records no reason, so it does not apply. The thread shows no case where pg_rewind or a standby's timeline choice depends on the old parent value; that cannot be checked without the code.
- security: 0 objection(s). The proposal changes only the parent timeline that the server writes into a history file at the end of archive recovery. It adds no new privilege, SQL surface, setting or exposure, and no unprivileged user can reach or influence this path, so there is no security angle to review.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is nothing to object to. The patch changes the parent argument of one existing writeTimeLineHistory() call so that it matches the PrevTimeLineID already written to the end-of-recovery record. It adds no GUC, catalog change, API, dependency or hot-path work, and later features have no new place to update. The bug belongs in core because the history file is a core format. Three reviewers agree on the design (messages #2, #3, #6). Committability: the committers who know this area were asked in message #2 and have not replied. The back-patch case would be stronger with a concrete broken consumer, which message #6 asks for. One candidate from general knowledge: a standby on timeline 1 between A and B may follow timeline 3 because the wrong file shows timeline 1 valid until B. The patch also changes promotion so that it no longer restores the target timeline's history file from the archive, as message #6 shows for 002_archiving.pl. External archive tools that expect that file could notice this, but the material shows no such tool. [P11] is about pg_rewind and has no recorded reason, so it does not apply.
- already-exists: 0 objection(s). This is a bug fix to the content of the history file that core writes at end of recovery. No existing feature, option or extension can make core write a correct parent timeline, so the already-exists angle has nothing to raise. [P11] is about pg_rewind and diverging timelines, and no reason is recorded for it, so it does not apply.
- demand: 0 objection(s). This is a small correctness fix with no ongoing cost and no new user surface. It also makes the history file agree with the PrevTimeLineID that the end-of-recovery record already uses. Message #6 asks for user-visible impact, but that only decides whether to back-patch, not whether to fix master, and I found no concrete failure scenario to support a demand objection.

### Precedents retrieved

- P1: [get current and recovery timeline](https://commitfest.postgresql.org/patch/2225/) (cf, Returned with feedback, 2020-01-31) - judged not relevant; reason summarised from its thread
- P2: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P3: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P4: [New argument "latest" for recovery_target parameter](https://commitfest.postgresql.org/patch/2347/) (cf, Returned with feedback, 2020-02-01) - judged not relevant; reason summarised from its thread
- P5: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P6: [Creating backup history files for backups taken from standbys](https://commitfest.postgresql.org/patch/1231/) (cf, Returned with feedback, 2018-03-05) - judged not relevant; reason summarised from its thread
- P7: [Allow subfield references without parentheses](https://commitfest.postgresql.org/patch/5449/) (cf, Returned with feedback, 2025-02-14) - judged not relevant; reason summarised from its thread
- P8: [history of vacuum / checkpoint runs](https://commitfest.postgresql.org/patch/5462/) (cf, Withdrawn, 2025-11-14) - judged not relevant; reason summarised from its thread
- P9: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - judged not relevant
- P10: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant; reason summarised from its thread
- P11: [pg_rewind does not rewind diverging timelines](https://commitfest.postgresql.org/patch/6811/) (cf, Withdrawn, 2026-05-29) - used in the review
- P12: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P13: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P14: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P15: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant
- P16: [PITR enhance getRecordTimestamp()](https://commitfest.postgresql.org/patch/3222/) (cf, Withdrawn, 2022-09-16) - judged not relevant
- P17: [Make message at end-of-recovery less scary](https://commitfest.postgresql.org/patch/2490/) (cf, Withdrawn, 2024-01-31) - judged not relevant
- P18: [PITR shutdown should not report error by pg_ctl](https://commitfest.postgresql.org/patch/4653/) (cf, Withdrawn, 2023-11-30) - judged not relevant
- P19: [recovery modules](https://commitfest.postgresql.org/patch/4082/) (cf, Withdrawn, 2024-07-19) - judged not relevant
- P20: [SQL function which allows to distinguish a server being in point in time recovery mode and an ordinary replica](https://commitfest.postgresql.org/patch/5513/) (cf, Withdrawn, 2025-03-17) - judged not relevant
- P21: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P22: [WALWriter during Recovery](https://commitfest.postgresql.org/patch/46/) (cf, Returned with feedback, 2015-09-01) - judged not relevant
- P23: [Reduce the time required for a database recovery from archive.](https://commitfest.postgresql.org/patch/2732/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P24: [SQL function which allows to distinguish a server being in point in time recovery mode and an ordinary replica](https://commitfest.postgresql.org/patch/5611/) (cf, Returned with feedback, 2025-08-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 102 s; cost reported by the CLI: $0.94
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:29:20.121Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
