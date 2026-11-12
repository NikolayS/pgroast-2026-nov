# Design pre-review: Add archive_mode to prevent unarchived WAL on permanent primary failure

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6177](https://commitfest.postgresql.org/patch/6177/) (Needs review); authors: Andrey Borodin (x4m), Kirill Reshke (reshke)
- Thread: [Streaming replication and WAL archive interactions](https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com), started 2014-12-12
- Messages read: 42 of 42
- Patch prose read: 0001-Make-WAL-archival-behave-more-sensibly-in-standby-mo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Several operators report the PITR gap, so the goal is not in doubt. The design has open safety and retention points: replication-ondisk-3 (ancestor-timeline segments marked .done without proof), replication-ondisk-1 (no bound on standby WAL retention) and prior-attempts-1 (no archive_command contract for duplicates). already-exists-1 and already-exists-2 ask what shared mode saves over always mode or a smaller first step. The list needs to settle these before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The v8 code was not read. Whether 'a' is sent only on request, how the fallback timer works, and how shared mode behaves when the standby restores from the archive could not be checked.

## The proposal, restated

**Problem.** With streaming replication and a shared WAL archive, a promoted standby does not archive segments it received by streaming, so WAL the old primary never archived is missing and PITR has gaps. Andrey Borodin reports gaps in 1–2% of clusters after large datacenter outages. HA tools that re-archive from the standby fail when the standby has already removed those segments.

**Proposed design.** A new archive_mode value, shared, makes the standby keep .ready status for streamed segments and mark them .done only when the upstream reports, through a new replication message, that it archived them. On promotion the standby archives what the primary did not. Cascading standbys relay the report downstream, and v8 adds a negotiated request, a FATAL on unarchiving upstreams and a timed fallback archiving on standbys.

**User-visible surface.**
- New archive_mode value shared, documented in a new "Continuous archiving in standby" section.
- New walsender message Byte1('a') carrying the last archived WAL segment.
- New GUC archive_status_report_interval (default 10s), which the standby also uses to request reports in v8.
- New column primary_last_archived in pg_stat_wal_receiver.
- Connection rejected with FATAL when the standby requests reports and the upstream does not archive (v8-0003).

**What the author claims.**
- archive_mode = always with an archive tool costs too much; in WAL-G it raises archive storage requests about 2.5 to 3 times.
- Shared mode solves PITR gaps inside Postgres instead of by distributed coordination in archive tools.
- v4 ran in production for about 3 to 4 months and was mostly fine, apart from the cascading and restartpoint bugs fixed later.
- Several large operators report the same problem and want at least last archived WAL exposed on the standby.
- Most users run a shared archive between primary and standbys (Borodin Vladimir, 2014).

**Thread state.** Heikki's 2014–2015 shared mode stalled and only always was committed; Andrey Borodin revived it in 2026, and Kirill Reshke posted v6 to v8. Andrey's last review of v8 lists open issues: fallback timing after idle periods, reports disabled by wal_sender_timeout = 0, ancestor-timeline .done marking, docs, and Windows test paths.

## 2. Prior attempts

### Better fix for "unarchived WAL files get deleted on crash recovery" bug. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145)
- **Date:** decided 2013-02-15; started 2013-02-15
- **Outcome:** Partially reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert my earlier fix for the bug that unarchived WAL files get deleted on crash recovery, commit c9cc7e05c6d82a9781883a016c70d95aa4923122. We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery. Per Fujii Masao's suggestion.
- **Reason (quote):** "We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery."
- **What is different this time:** P4 set the rule that streamed and restored segments get .done so normal recycling works in archive recovery. Shared mode reverses this for streamed segments and keeps them .ready, which needs its own recycling guarantees.


## 3. Objections a reviewer is likely to raise

### 1. Ancestor-timeline segments marked .done without proof they are archived

fixable · level 2, rubric item 7 · WAL / replication / standby · `replication-ondisk-3`

After a timeline switch the standby receives a report for the descendant timeline. It marks the ancestor's switchpoint segment .done and recycles it, but the archive holds only the descendant-named file. A later PITR with recovery_target_timeline set to the ancestor cannot find that segment, and the gap stays hidden until a restore is attempted.

Message #42 raises this as open: a report on a descendant timeline proves only the common prefix under the descendant filename. The feature exists to remove PITR gaps, so marking a file .done that may be absent from the archive breaks the invariant the mode promises. Messages #21 to #25 show that the switchpoint segment naming is subtle and that recovery depends on it.

Thread: [#21 (Heikki Linnakangas, 2015-04-22)](https://www.postgresql.org/message-id/5537EFEF.6060808%40iki.fi), [#25 (Heikki Linnakangas, 2015-04-23)](https://www.postgresql.org/message-id/5538A927.7000407%40iki.fi), [#42 (Andrey Borodin, 2026-07-31)](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru)

**Question:** What guarantees that the ancestor-named switchpoint segment is in the archive before the standby marks it .done, and which archive-only recovery test shows it?

Counter-check: held. Message #42 raises this as open and asks for an explanation or an archive-only recovery test. Nothing later in the visible thread answers it.

### 2. Standby WAL retention has no bound when archive reports stop

fixable · level 2, rubric item 14 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

The archive storage goes down for a day while the primary keeps writing WAL. Every shared-mode standby keeps all streamed segments as .ready and fills pg_wal, then stops. After failover the next standby is in the same state, so the whole cluster loses availability because of a backup outage.

Message #40 reports this from production and says the downstream fork needed a SIGHUP escape hatch, because archive_mode is PGC_POSTMASTER. v8 (message #41) handles an upstream that is configured not to archive, through the FATAL and the fallback. It does not handle an upstream that is configured to archive but cannot. No cap similar to max_slot_wal_keep_size is proposed, and the docs do not describe this failure mode.

Thread: [#40 (Andrey Borodin, 2026-07-22)](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** What limits pg_wal growth on a shared-mode standby while archive_command fails on the primary, and how does an operator release that WAL without a restart?

Counter-check: held. Message #40 point 7 reports this from production, and the v8 fallback does not help because the standby's archive_command fails against the same unavailable archive. No cap or runtime release is described in message #41.

### 3. Duplicate archival, which stalled shared mode in 2015, is still unresolved

fixable · level 2, rubric item 7 · prior rejection, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

In a v8 setup the standby runs fallback archiving after a quiet period while the primary archives the same segment. An archive_command that refuses to overwrite, as the manual recommends, returns failure. The archiver then retries that segment forever and WAL piles up in pg_wal until someone notices the disk filling.

In message #29 Heikki confirmed that a promoted shared-mode standby gets stuck retrying segments the primary had already archived. In message #31 he cut the patch back to always, because the fix needs a duplicate-tolerant command. Message #20 asked the project not to add more cases that users must "get right" in archive_command. The v8 fallback (message #41) makes double archival routine, and message #42 shows its timing can race the primary, yet the proposal does not yet define the archive_command contract for identical files.

Thread: [#20 (Robert Haas, 2015-04-22)](https://www.postgresql.org/message-id/CA%2BTgmoY7wkYi-27LOnm128RRw8cQ%3DtNhJdPGgJ0JgpQZ-uwUuA%40mail.gmail.com), [#29 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#30 (Robert Haas, 2015-05-13)](https://www.postgresql.org/message-id/CA%2BTgmoY%2Bp5oowv1%3DZ44Zu14V70XKQZBkt%2BoEmwB%2BgAYYfDAV1Q%40mail.gmail.com), [#31 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com), [#42 (Andrey Borodin, 2026-07-31)](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru)

**Question:** What exact archive_command behavior does shared mode require for a segment already archived, and how does the archiver avoid looping on it with the documented example command?

Counter-check: held. Message #41 says primary and cascade standby both archive under the v8 fallback, and message #29 confirms a no-overwrite command gets stuck on a duplicate. The visible thread does not state what archive_command must do with an identical existing file.

### 4. archive_mode = always with a duplicate-tolerant archive_command already closes the gap

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), wire protocol, new GUC, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator who runs archive_mode = always on standbys with a shared archive and a duplicate-tolerant archive_command already gets a gap-free archive after promotion. Shared mode adds a second archiving mode, a replication message, a GUC and fallback logic for the same outcome. Every later change to archiving or walsender then has two cooperating paths to keep correct.

In message #31 Heikki dropped shared mode, saying always plus a race-free archive_command that handles duplicates makes it unnecessary. The only answer is cost: about 2.5 to 3 times more archive storage requests in WAL-G (messages #32, #39). That makes always more expensive, not impossible. Shared mode does not remove the duplicate requirement either: promotion re-archives what the primary may already have stored (message #29), and v8 fallback lets primary and standby archive the same segment (message #41).

Thread: [#29 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#31 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#32 (Andrey Borodin, 2026-02-12)](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#39 (Kirill Reshke, 2026-07-22)](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** Since shared mode still needs a duplicate-tolerant archive_command for promotion and fallback, which setups cannot use archive_mode = always, beyond the extra storage requests?

Counter-check: held. Three operators report the gap problem, so the need is common, but the only stated case against always mode is request cost. Messages #29 and #41 show shared mode still needs a duplicate-tolerant archive_command, and the thread does not say what it then saves.

### 5. Exposing the upstream archive position may be enough for external tools

fixable · level 1b, rubric item 11 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), wire protocol · `already-exists-2`

Core takes on WAL retention, cascading relay, request negotiation, a FATAL on connect and timed fallback archiving. The open v8 issues show these are hard to get right: idle-period races, wal_sender_timeout = 0, ancestor timelines. HA tools would have managed with a queryable primary_last_archived and a bounded retention setting.

In message #33 a second operator says that exposing last_archived_wal on the standby alone "would unblock everyone solving this problem externally". That is the 0002 piece. Standby-side retention does need core help, and a committer would likely ask whether that help can stay small. One candidate is a primary_last_archived column plus a retention rule, with promotion-time archiving left to the HA tool, which already re-archives from the standby (message #32).

Thread: [#32 (Andrey Borodin, 2026-02-12)](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#33 (Harinath Kanchu, 2026-02-20)](https://www.postgresql.org/message-id/41D301A5-AB81-4B21-8AC1-9DD602362D31%40apple.com), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com), [#42 (Andrey Borodin, 2026-07-31)](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru)

**Question:** What would HA tools still be unable to do if core only exposed primary_last_archived and held back recycling of unconfirmed segments, without fallback archiving?

Counter-check: held. Message #33, from an independent operator, says exposing the last archived segment on the standby would be enough for external solutions. The thread does not explain why the smaller piece, with or without a retention rule, cannot go first.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Shared mode changes when the standby creates .ready and .done files and when it recycles segments in pg_wal. |
| WAL / replication / standby | **yes** | The standby keeps streamed WAL until the upstream reports it archived, and cascading standbys relay the report. |
| wire protocol | **yes** | The patch adds walsender message Byte1('a') and, in v8, a request for it from the standby. |
| catalog changes | **yes** | The patch adds column primary_last_archived to the pg_stat_wal_receiver view. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds archive_status_report_interval and a new archive_mode value, shared. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The mode spans archiver, walsender, walreceiver and recovery, with relay, negotiation, FATAL and timed fallback states. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | archive_mode = always or an exposed primary_last_archived with external tools may cover the same need. |

## 5. Questions the author must answer before anyone reviews code

1. Since promotion and the v8 fallback still need a duplicate-tolerant archive_command, which setups cannot use archive_mode = always, beyond the extra storage requests?
2. What guarantees that the ancestor-named switchpoint segment is in the archive before the standby marks it .done, and which archive-only recovery test shows it?
3. What limits pg_wal growth on a shared-mode standby while archive_command fails on the primary, and how does an operator release that WAL without a restart?
4. What must archive_command do with a segment that is already in the archive, and how does the documented no-overwrite example avoid an endless retry?
5. In v8, does any walsender send 'a' to a client that did not request it, and how does an older pg_receivewal behave against a shared-mode primary?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the archiver, walsender and walreceiver, and timeline handling in recovery. The 2014 design came from a committer, but none has commented on v6 to v8.
- **Liability:** The owner signs for a permanent archive_mode value, a replication protocol message and a GUC. The owner also signs for PITR completeness across timeline switches, where a mistake shows only at restore time.
- **Author follow-up:** The authors posted v6 to v8, carried the entry through six CommitFests, and report running v4 in production for about 3 to 4 months.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Several operators with shared archives report PITR gaps, so the goal has users. The patch adds a protocol message, a GUC and new recovery semantics, and the co-author's own review of v8 lists open design issues that should close before a committer spends time.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Unsolicited 'a' walsender message breaks existing physical protocol clients. Not listed: not among the strongest objections.
- `standards-compat-3` (fixable, level 2) archive_status_report_interval combines three roles and can reject replication connections. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `prior-attempts-2` The 2015 conclusion that always mode suffices is not yet answered. Refuted (not specific): Duplicate of already-exists-1 for the always-mode point and of already-exists-2 for the smaller-piece point. Message #31 also says shared mode "might still make sense as a future patch", so it is not a conclusion against it.
- `standards-compat-2` Fallback archiving changes the archive_command contract without documenting it. Refuted (not specific): Duplicate of prior-attempts-1, which states the same duplicate-archiving scenario and the same missing archive_command contract.
- `replication-ondisk-2` Fallback archiving brings back duplicate archiving that shared mode was meant to avoid. Refuted (not specific): Duplicate of prior-attempts-1. The added point about lighter archive_command requirements than always mode is covered by already-exists-1.
- `replication-ondisk-4` Unsolicited 'a' replication message may still break other physical protocol clients. Refuted (not specific): Duplicate of standards-compat-1. The protocol.sgml hunk it cites is from the 2014 patch in message #5, so it says nothing about v8.
- `maintenance-fit-1` Timed fallback brings back duplicate archiving that shared mode was meant to avoid. Refuted (not specific): Duplicate of prior-attempts-1. Its last sentence about each new state diverging silently names no concrete case.
- `maintenance-fit-2` Standby WAL retention has no bound and no runtime escape. Refuted (not specific): Duplicate of replication-ondisk-1, which gives the same scenario and also notes that v8 covers only an upstream configured not to archive.
- `maintenance-fit-3` No committer engaged since the 2026 revival. Refuted (not specific): The absence of an engaged committer is a committability note and, under the rubric, does not count against the design. Its failure scenario is the one in replication-ondisk-3, and a committer wrote the original shared-mode design.
- `maintenance-fit-4` Smaller observability step is not offered as a separate first commit. Refuted (not specific): Duplicate of already-exists-2, which asks the same question and also grants that standby-side retention needs core support.

### Review angles

- prior-attempts: 2 objection(s). The thread contains its own prior attempt, Heikki's 2014-2015 shared mode (messages #5 to #31). Only always mode was committed, and the 2026 work revives that shared mode.
- standards-compat: 3 objection(s). The objections cover wire protocol, archive_command contract and GUC semantics. Unbounded WAL retention is left to the safety angle.
- replication-ondisk: 4 objection(s). Objections stay inside replication and persistent state. Behaviour when the standby switches from streaming to restore_command (message #4, still open in message #39) could not be judged from the material.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. The 'a' message travels only over replication connections, which already need the REPLICATION privilege, and it carries the last archived segment name, which pg_stat_archiver already shows. The primary_last_archived column has the same low sensitivity. Unbounded WAL retention on a standby comes from an archive outage or from configuration set by the administrator, and no unprivileged user can trigger it. The other parts, the standby's fallback archive_command and duplicate archiving, use commands the administrator sets. Their risks are about archive integrity and availability, which other angles cover. One point could not be checked without the code: whether primary_last_archived follows the existing pg_read_all_stats filtering in pg_stat_wal_receiver.
- maintenance-fit: 4 objection(s). Objections were raised. Wire-protocol breakage of pg_receivewal is left to other angles.
- already-exists: 2 objection(s). Two objections from the already-exists angle; no listed precedent applies.
- demand: 0 objection(s). The material shows demand from several independent operators. Message #32 reports PITR gaps in 1-2% of clusters after datacenter outages, message #37 describes tens of thousands of clusters that work around the problem by external synchronization, and message #33 reports the same production problem from a separate group. Messages #1 and #2 describe a shared archive as a common setup. The workaround in use today, archive_mode = always, is said to cost about 2.5 to 3 times more archive storage requests (message #39). Message #33 says exposing the upstream's last archived WAL on the standby would already help, so a narrower first step may be worth discussing. That is a scope and design question, not missing demand.

### Precedents retrieved

- P1: [Track number of WAL files ready to be archived in pg_stat_archiver](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Statistics_Collector) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P2: [Revert addition of poorly-thought-out DUMP TIMESTAMP archive entry, which induced bug #1597 in addition to having several other misbehaviors (like labeling...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=348f856dc57902a8f951a570284918761da1776b) (revert, Reverted, 2005-04-15) - judged not relevant
- P3: [Allow archive_mode to be changed without server restart?](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Point-In-Time_Recovery_.28PITR.29) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P4: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - used in the review
- P5: [Revert "Correct WAL space calculation formula in docs."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cae0aeb9c61aef48b6de3fb5c4f7785a06bca0b) (revert, Reverted, 2010-10-28) - judged not relevant
- P6: [Change walsender so that it applies per-role settings](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P8: [Revert "Add mode where contrib installcheck runs each module in a separately named database."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fc5c1bbbeb3e00699f87407eebf2933595e9d93b) (revert, Reverted, 2012-12-03) - judged not relevant
- P9: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P10: [Allow pg_xlogfile_name() to be used in recovery mode](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Revert 1fbc30f1dca6624c01a9a6a9ff711a7036c68532 because the --enable-shared is about Plyton's configure, not ours.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39d0653d058b87b437be05fd295ceb5bb4e9e05c) (revert, Reverted, 2012-08-30) - judged not relevant
- P12: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P13: [Revert part of recent include patch not ready for application.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=51e888219745cf9efa41034b5411eee7e9a8f4f1) (revert, Reverted, 2006-07-14) - judged not relevant
- P14: [Revert to the pre-8.2 method of probing for libm, that is, always include it if it links properly.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=02f37bd8c4bfdd35869007d142267cd9a1f671b9) (revert, Reverted, 2006-11-06) - judged not relevant
- P15: [Revert error message on GLOBAL/LOCAL pending further discussion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28ac7972873bd314d4837807396fe44571b5cb50) (revert, Reverted, 2012-06-10) - judged not relevant
- P16: [Revert dubious message wording change.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1c8f03d4de5bca70437aaff5548a18a2d4e16a11) (revert, Reverted, 2009-06-22) - judged not relevant
- P17: [Revert "Auto-tune effective_cache size to be 4x shared buffers"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a16d421ca4fc639929bc964b2585e8382cf16e33) (revert, Reverted, 2014-05-09) - judged not relevant
- P18: [Revert "Allow units to be specified in relation option setting value."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9df492664a74a948c87232f523eea637920a516b) (revert, Reverted, 2014-08-28) - judged not relevant
- P19: [Revert previous patch to put the shared memory segment on win32 in the Global\ namespace, because it caused permission errors on a lot of platforms.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9cc0222680229c4572354cf2b3907c7ff18ad685) (revert, Reverted, 2008-10-30) - judged not relevant
- P20: [Revert all keepalive moves, reevaluate.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54aaef864213d7b91f1efc691d3dc93ba468815b) (revert, Reverted, 2010-03-13) - judged not relevant
- P21: [Revert error message change for may/can/might --- needs discussion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=482e6936fa2063cd18824f15b3edd2b383cc0009) (revert, Reverted, 2007-01-31) - judged not relevant
- P22: [Revert to our pre-7.4 behavior of identifying Unix-socket connections in ps status as '\[local\]', not as 'localhost' as the code has been doing recently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b43e325ebfab423c937ce384bcdaf70ed0b462e) (revert, Reverted, 2003-09-12) - judged not relevant
- P23: [Revert off setting mention for client_min_messages.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7af352d0913b77f387a489cd79b54b040cbee7e8) (revert, Reverted, 2003-01-28) - judged not relevant
- P24: [revert last change](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=60e42602a01b710b9489fb679405f5e21eea8704) (revert, Reverted, 2001-12-13) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com). Others: https://www.postgresql.org/message-id/flat/EBB5D20D-8630-4557-999F-999BAD90BCCF%40yandex-team.ru

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 158 s; cost reported by the CLI: $4.09
- Precedent enrichment: 1 thread(s) lacked a recorded reason: 1 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 126 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:44:14.866Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
