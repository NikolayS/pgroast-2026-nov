# Design pre-review: pg_rewind: file sync bypass and findLastCheckpoint boundary crash

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7221](https://commitfest.postgresql.org/patch/7221/) (Needs review); authors: Srinath Reddy (srinath-reddy-sadipiralla)
- Thread: [\[BUG\] pg_rewind: file sync bypass and findLastCheckpoint boundary crash](https://www.postgresql.org/message-id/flat/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com), started 2026-08-30
- Messages read: 2 of 2
- Patch prose read: v1-0001-Fix-pg_rewind-file-sync-bypass-and-findLastCheckpoin.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

Level 1b is where a committer would stop. already-exists-1 says a cleanly stopped old primary started as a standby follows the new timeline today, and the thread shows only a postgresql.auto.conf difference. standards-compat-1 says the planned switchover changes from no writes to a file copy, and the thread does not weigh that. Only the author has posted, so the verdict rests on what the thread does not yet show.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, I could not check how the fallback detects end of written WAL, where the forward scan stops, or which files the sync copies in this case. The linked thread from message #2 is not in the material.

## The proposal, restated

**Problem.** When a primary shuts down cleanly and a standby is promoted, the target's WAL ends exactly at the divergence point, and pg_rewind exits with no rewind needed. The target then lacks the new timeline history file and non-WAL-logged changes such as postgresql.auto.conf, and pg_control keeps the old timeline and state. This affects anyone who rewinds a cleanly stopped old primary.

**Proposed design.** The patch removes the early exit for target_wal_endrec == divergerec, so pg_rewind runs the file sync whenever timelines differ. That exposes a failure in findLastCheckpoint(), whose backward read at divergerec hits unwritten WAL. When that first backward read fails at the fork point, findLastCheckpoint() falls back to a forward scan from the pg_control checkpoint LSN to find the last common checkpoint.

**User-visible surface.**
- pg_rewind no longer reports that no rewind is needed when target WAL ends at the divergence point; it copies files and updates pg_control instead.

**What the author claims.**
- The early exit leaves the target without the source's timeline history file, non-WAL-logged file changes and an updated pg_control.
- Removing the early exit alone makes pg_rewind fail with "invalid record length ... expected at least 24, got 0".
- The forward-scan fallback finds the last common checkpoint correctly in this case.
- A new TAP test, using ALTER SYSTEM right after promotion, shows the file-sync failure on master.

**Thread state.** Only the author has posted; the second message just adds the missing link to the related thread. No reviewer has replied, and CFbot reported that the patch needs a rebase.

## 2. Prior attempts

### Fix pg_rewind false positives caused by shutdown-only WAL (related)

- **Link:** [https://commitfest.postgresql.org/patch/6041/](https://commitfest.postgresql.org/patch/6041/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com)
- **Date:** decided 2026-05-01; started 2025-09-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas objected to the premise. The shutdown checkpoint updates the control file, and that change matters when deciding whether servers are in sync. Even if pg_rewind ignored those records, the restarted old primary still refuses to follow the new timeline. He also noted that the patch did not skip the RUNNING_XACTS record and did not check the rmgr ID, and he saw no workaround; a later patch from another poster got no reply before the entry was withdrawn.
- **Reason (quote):** "But I do not think we can do that, because (1) those WAL records might do important things like update the control file and (2) the server will not be OK with ignoring those WAL records even if pg_rewind decides that they are not important." ([message](https://www.postgresql.org/message-id/CA%2BTgmoYYvYShoRKK6%3D8PX-BYBarFXjrGB%2BcBBRjX-SkMfd0Nrg%40mail.gmail.com))
- **What is different this time:** P8 made pg_rewind skip shutdown-only WAL so that fewer cases need a rewind. This patch does the opposite: it runs the file sync and updates pg_control in a case that used to exit early, which agrees with P8's objection.


## 3. Objections a reviewer is likely to raise

### 1. Clean switchover already works by starting the old primary as standby

fixable · level 1b, rubric item 2 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), WAL / replication / standby · `already-exists-1`

After a clean switchover, an operator adds standby.signal and primary_conninfo to the old primary and starts it, and it follows the new timeline. With the patch, a pg_rewind run in the same state copies files and rewrites pg_control through a new forward-scan path. A regression there breaks a switchover that needed no rewind at all.

In this case the old primary's WAL ends at the divergence point, so it can follow the new timeline. Recovery fetches the timeline history file from the new primary or the archive and switches timelines itself. Message #1 lists the missing history file, pg_control state and postgresql.auto.conf, but the first two are already handled by recovery. The third is a configuration difference, and standbys often keep their own settings on purpose. So the gap is convenience in tooling that always runs pg_rewind, and the material does not show anything that cannot be done today.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com)

**Question:** What concrete failure does a user hit when the old primary is started as a standby without pg_rewind after a clean switchover, which recovery cannot already handle?

Counter-check: held. A cleanly stopped old primary started with standby.signal fetches the history file and follows the new timeline, and recovery updates pg_control. The thread shows only a postgresql.auto.conf difference, so it shows no failure that needs the file sync.

### 2. Planned switchover now rewrites the old primary's files and pg_control

fixable · level 1b, rubric item 6 · desirability, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An HA tool stops the primary cleanly, promotes a standby, then runs pg_rewind as a routine step. Today pg_rewind reports that no rewind is required and leaves the old primary alone. After the patch it overwrites postgresql.auto.conf and other config files with the new primary's copies and rewrites pg_control, and operators find out when the old primary starts with settings they did not choose.

A clean stop followed by promotion is the normal planned switchover, so this is the common path. Message #1 treats copying postgresql.auto.conf as the desired result. But in this case some setups may rely on pg_rewind leaving the old primary untouched. The patch touches no documentation files, so neither the pg_rewind docs nor any release note would describe the change.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com)

**Question:** Which tools or scripts run pg_rewind after a clean switchover, and does each of them expect the old primary's config files to be overwritten?

Counter-check: held. pg_rewind already copies config files in every real rewind, so tools that run it must cope with that today. But the clean-switchover case changes from no writes to a file copy, and the thread does not weigh that change.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_rewind now rewrites the target's pg_control and copies files in a case where it wrote nothing before. |
| WAL / replication / standby | **yes** | The patch changes how findLastCheckpoint() reads target WAL and how an old primary is prepared to follow a new timeline. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix but changes visible pg_rewind behaviour; it sets target version 20 and does not mention back branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | findLastCheckpoint() gains a second, error-triggered forward-scan mode that later xlogreader changes must keep working. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Starting the cleanly stopped old primary as a standby may already cover this case without pg_rewind. |

## 5. Questions the author must answer before anyone reviews code

1. What concrete failure does a user hit when a cleanly stopped old primary starts as a standby without pg_rewind, which recovery cannot already handle?
2. Was the early exit for target_wal_endrec == divergerec intended, and do any tools rely on pg_rewind leaving the target untouched in that case?
3. Which files does the sync copy in this case, and why is copying the source's postgresql.auto.conf over the target's the wanted result?
4. Why not take the last common checkpoint from the target's pg_control when target_wal_endrec equals divergerec, instead of adding a forward scan to findLastCheckpoint()?
5. How does the fallback tell end of written WAL from a missing or damaged segment at forkptr, and what happens if the forward scan finds no checkpoint?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows pg_rewind, timeline switching and the xlogreader code. Only the author has posted in the thread.
- **Liability:** The owner signs for pg_rewind's behaviour on the planned-switchover path, which HA tools run routinely, and for the forward-scan path in findLastCheckpoint() for as long as it exists.
- **Author follow-up:** No clear sign. The author posted twice on the same day, and the rebase that CFbot asked for on 2026-09-15 is not visible in the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch is small and touches only pg_rewind, but it changes the result of the common planned-switchover case and reads WAL in a new way. The thread must first show what fails today and whether the checkpoint can come from pg_control.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-1` (fixable, level 2) Error-triggered fallback in findLastCheckpoint where the checkpoint is already known. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-2` Bug-fix tag implies back-patching a behaviour change. Refuted (no failure scenario): The scenario needs a back-patch, and the proposal does not include one. The CommitFest entry sets target version 20, and the behaviour change on master is covered by standards-compat-1.
- `replication-ondisk-1` Clean-shutdown case goes from no writes to a full rewind. Refuted (not specific): Duplicate of already-exists-1 for the main question, which is what fails without a rewind. The interrupted-copy risk is the same one every pg_rewind run already has, and the copy is of changed files, so it is smaller than a full rewind.
- `replication-ondisk-2` Forward-scan fallback has no stated trigger or result check. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same trigger question and names the pg_control alternative. The claim that the scan picks a checkpoint without checking it cannot be verified from the patch prose.
- `replication-ondisk-3` Behaviour change in a bug-fix entry affects HA tools if back-patched. Refuted (not specific): Duplicate of standards-compat-1 for the effect on HA tools. The back-patch part depends on a step nobody proposed, since the entry sets target version 20.
- `demand-1` No user-visible failure shown for the clean switchover case. Refuted (not specific): Duplicate of already-exists-1, which asks the same question about what fails without a rewind and gives the recovery mechanism in more detail.
- `demand-2` Every clean switchover now gets config files overwritten. Refuted (not specific): Duplicate of standards-compat-1. Both describe config files overwritten on a clean switchover that today leaves the target untouched.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected for a reason that applies here. P8's objection, that control-file state matters when deciding whether servers are in sync, supports this patch's premise. P7 and P1 cover other pg_rewind cases (cascading source, same-TLI forks), and their reasons do not carry over. Message #1 says the bug was found while working on another thread, and message #2 links it. The material does not show what that thread contains.
- standards-compat: 2 objection(s). Two objections from the compatibility angle. No new syntax, GUC, catalog object or psql change is proposed.
- replication-ondisk: 3 objection(s). Three objections are listed. Without the code I could not check how the fallback decides it is at end of WAL, or which files the sync copies in this case.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. pg_rewind is an operator tool that runs with access to the target data directory and the source connection or directory, and the patch only makes it run its normal file sync in one more case, with no new privilege, role, setting or path for unprivileged users.
- maintenance-fit: 1 objection(s). Only one maintenance objection, and it is fixable. The change is confined to pg_rewind, adds no hot-path cost, GUC, catalog or external dependency, and fixes a real gap. No committer has engaged yet, which bears on committability rather than merit.
- already-exists: 1 objection(s). One objection from this angle. The findLastCheckpoint() fallback only matters once the early exit is removed, so it stands or falls with the objection.
- demand: 2 objection(s). Two objections are listed.

### Precedents retrieved

- P1: [pg_rewind does not rewind diverging timelines](https://commitfest.postgresql.org/patch/6811/) (cf, Withdrawn, 2026-05-29) - used in the review; reason summarised from its thread
- P2: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant; reason summarised from its thread
- P3: [Make pg_rewind not copy useless WALs](https://commitfest.postgresql.org/patch/1302/) (cf, Returned with feedback, 2018-01-31) - judged not relevant; reason summarised from its thread
- P4: [Data-only mode for pg_rewind](https://commitfest.postgresql.org/patch/2063/) (cf, Returned with feedback, 2019-08-01) - judged not relevant; reason summarised from its thread
- P5: [Add tablespace tap test to pg_rewind](https://commitfest.postgresql.org/patch/2059/) (cf, Returned with feedback, 2019-05-10) - judged not relevant; reason summarised from its thread
- P6: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P7: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P8: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - used in the review; reason summarised from its thread
- P9: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P10: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P11: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P12: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - judged not relevant
- P13: [Revert "Switch TAP tests of pg_rewind to use a role with minimal permissions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=db8db624e826efbe16aab1ae921bae071f98f099) (revert, Reverted, 2019-04-13) - judged not relevant
- P14: [Index Skip Scan](https://commitfest.postgresql.org/patch/1741/) (cf, Returned with feedback, 2022-03-24) - judged not relevant
- P15: [Tid range scan](https://commitfest.postgresql.org/patch/1755/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P16: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant
- P17: [num_sa_scans in genericcostestimate](https://commitfest.postgresql.org/patch/3871/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P19: [Allow providing restore_command as a command line option to pg_rewind](https://commitfest.postgresql.org/patch/3213/) (cf, Rejected, 2022-04-07) - judged not relevant
- P20: [Optimization for lazy_scan_heap](https://commitfest.postgresql.org/patch/699/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P21: [Scan key push down to heap](https://commitfest.postgresql.org/patch/850/) (cf, Returned with feedback, 2016-12-21) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 170 s; cost reported by the CLI: $1.29
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 21 kept
- Tone check: 91 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:30:41.844Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
