# Design pre-review: Add archive_mode to prevent unarchived WAL on permanent primary failure

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6177](https://commitfest.postgresql.org/patch/6177/) (Needs review); authors: Andrey Borodin (x4m), Kirill Reshke (reshke)
- Thread: [Streaming replication and WAL archive interactions](https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com), started 2014-12-12
- Messages read: 42 of 42
- Patch prose read: 0001-Make-WAL-archival-behave-more-sensibly-in-standby-mo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is stated and three fleet operators report it, so level 1 holds. At level 2, four design points are open in v8: the archive_command contract (prior-attempts-1), the 'a' message reaching clients that did not ask (standards-compat-1), unbounded WAL retention (maintenance-fit-3) and ancestor timeline marking (replication-ondisk-3). Shared mode was set aside in message #31 in favour of always mode, so the list also needs to agree on already-exists-1. All of these are fixable.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read, and the documentation hunks shown are from the 2015 patch, so v8 behaviour is known only from thread messages. The opt-in rule for 'a', the fallback timing and the timeline marking could not be checked.

## The proposal, restated

**Problem.** With streaming replication and one WAL archive for primary and standby, a promoted standby does not archive WAL it got by streaming. Segments the primary had not archived before it failed are missing from the archive, which breaks point-in-time recovery. Borodin reports gaps in the PITR timeline for 1–2% of clusters after large datacenter outages.

**Proposed design.** A new archive_mode value, shared, makes the standby mark streamed segments .ready and mark them .done only when the upstream reports them archived, so they are not removed earlier. The walsender sends the last archived segment in a new replication message, and cascading standbys relay it. On promotion the standby archives what the primary did not, and v8 adds a fallback where a standby archives itself when no reports arrive.

**User-visible surface.**
- archive_mode gains the value shared; Heikki's original patch also forbade on during recovery, and v8's handling of on is not stated.
- New replication protocol message 'a' (WAL archival report) carrying the last archived segment, sent by the walsender.
- New GUC archive_status_report_interval (default 10s) that sets report frequency; in v8 the standby also uses it to request reports.
- New column primary_last_archived in pg_stat_wal_receiver.
- Startup fails with FATAL when a standby requests reports and the upstream does not archive.

**What the author claims.**
- Shared mode fixes PITR gaps after losing the primary, seen in 1–2% of clusters in large outages.
- archive_mode = always with a deduplicating archive tool works, but would raise requests to the archive storage about 2.5 to 3 times.
- Fixing this in external HA tools fails because the standby may already have removed the WAL.
- v4 ran in production for about 3 to 4 months and was mostly fine, apart from the bugs fixed later.
- Several operators of large fleets (Harinath Kanchu, Grigory Smolkin) report the same problem and external workarounds.

**Thread state.** Heikki proposed shared mode in 2014 but committed only always and the .partial change in 2015; Borodin revived shared mode in 2026, and Reshke posted v6 to v8. Borodin's last review of v8 lists open items: fallback timing after idle periods, report scheduling, ancestor timeline marking, missing docs and Windows test paths.

## 2. Prior attempts

### Better fix for "unarchived WAL files get deleted on crash recovery" bug. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145)
- **Date:** decided 2013-02-15; started 2013-02-15
- **Outcome:** Partially reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** Revert my earlier fix for the bug that unarchived WAL files get deleted on crash recovery, commit c9cc7e05c6d82a9781883a016c70d95aa4923122. We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery. Per Fujii Masao's suggestion.
- **Reason (quote):** "We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery."
- **What is different this time:** P1 relies on creating .done for streamed segments so that recycling in recovery works as in normal operation. Shared mode creates .ready for them instead, and the v4 XLogArchiveCheckDone bug (messages #36, #38) came from that change.


## 3. Objections a reviewer is likely to raise

### 1. Shared mode still needs a duplicate-tolerant archive_command, the reason it was dropped

fixable · level 2, rubric item 12 · prior rejection, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

An operator follows the documented archive_command, which refuses to overwrite an existing file. The primary archives a segment and dies before its report reaches the standby, or a v8 fallback standby archives in parallel with the primary. After promotion the standby's archiver retries the same file indefinitely, .ready files pile up in pg_wal, and the operator learns of it when the disk fills.

In message #31, shared mode was set aside because always mode plus a race-free, duplicate-tolerant archive command does the job, and shared mode was kept only for its "less strict requirements for the archive_command". Messages #29 and #30 showed shared mode still needs that tolerance at promotion, and message #20 objected to adding more archive_command logic that users must get right. v8 widens the need, because its fallback (message #41) lets the primary and a cascading standby both archive. The author's stated gain over always mode is a 2.5-3x cut in archive requests (message #39), not a simpler archive_command.

Thread: [#20 (Robert Haas, 2015-04-22)](https://www.postgresql.org/message-id/CA%2BTgmoY7wkYi-27LOnm128RRw8cQ%3DtNhJdPGgJ0JgpQZ-uwUuA%40mail.gmail.com), [#29 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#30 (Robert Haas, 2015-05-13)](https://www.postgresql.org/message-id/CA%2BTgmoY%2Bp5oowv1%3DZ44Zu14V70XKQZBkt%2BoEmwB%2BgAYYfDAV1Q%40mail.gmail.com), [#31 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#39 (Kirill Reshke, 2026-07-22)](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** With the documented example archive_command, what happens after promotion or v8 fallback when the segment already exists in the archive, and does the patch ship a command that handles this?

Counter-check: held. Message #29 confirms the archiver gets stuck on a duplicate after promotion, and message #42 shows the v8 fallback can race the primary. The proposal names no archive_command contract for shared mode, and a new patch version or documentation can state one.

### 2. Unsolicited 'a' replication message breaks existing physical protocol clients

fixable · level 2, rubric item 12 · wire protocol, backpatchability · `standards-compat-1`

An operator runs pg_receivewal, or a third-party backup tool, with a permanent slot against a primary that has archive_mode = shared. The walsender sends an 'a' message, and the client stops with "unrecognized streaming header". With the retry loop it reconnects and fails again without progress, so the WAL copy silently stalls.

Message #40 reproduced this with pg_receivewal and noted that the PG 18 'p'/'s' pair is sent only on request, to avoid this hazard. Message #41 adds a standby-side request parameter in v8 but does not say whether the walsender still sends 'a' to clients that did not ask. Fixing receivelog.c covers in-core tools only. Older pg_receivewal binaries and external clients keep failing.

Thread: [#40 (Andrey Borodin, 2026-07-22)](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** In v8, does the walsender send 'a' only to clients that asked for it in START_REPLICATION, and what does an old pg_receivewal see?

Counter-check: held. Message #40 reproduced the pg_receivewal failure. Message #41 adds a standby-side request, which suggests opt-in, but no message says the walsender stopped sending 'a' to clients that did not ask.

### 3. No bound on WAL a shared-mode standby keeps

fixable · level 2, rubric item 14 · design, maintenance burden ("who maintains this in 10 years"), new GUC · `maintenance-fit-3`

The archive storage goes down for a day. Every shared-mode standby keeps all streamed WAL and fills its disk. Because archive_mode is PGC_POSTMASTER, the operator cannot turn retention off without a restart, and HA fails over to the next standby, which repeats the cycle.

Message #40 (point 7) reports this from production and says downstream added a SIGHUP GUC to escape it. Message #41's v8 handles only an upstream that never reports. It does not cap retention when reports stop because archiving stalls. Replication slots needed max_slot_wal_keep_size for the same class of problem.

Thread: [#40 (Andrey Borodin, 2026-07-22)](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** What limits WAL growth on a shared-mode standby during a long archive outage, and how does an operator release it without a restart?

Counter-check: held. Message #40, point 7, reports this from production, and v8 in message #41 covers only an upstream that never reports. A cap or a reloadable setting in a new version can fix it.

### 4. Marking ancestor switchpoint segment done may leave an archive gap

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, WAL / replication / standby · `replication-ondisk-3`

After a timeline switch, the standby marks the ancestor-timeline segment holding the switchpoint as .done because the upstream reported the descendant segment archived. If the ancestor filename was never archived, a later PITR with recovery_target_timeline set to the ancestor cannot find it.

Message #42 points out that a descendant report only proves the shared prefix exists under the descendant filename. No explanation or archive-only recovery test was given. This marks a segment as safe to recycle without proof, the gap the feature exists to prevent, and message #12 names PITR to the old timeline as a real use.

Thread: [#12 (Michael Paquier, 2015-04-21)](https://www.postgresql.org/message-id/CAB7nPqS%3DV%3DLF-JZsv9eGF84joQ3jxugt4gkOK6kD1CiB98_vWg%40mail.gmail.com), [#21 (Heikki Linnakangas, 2015-04-22)](https://www.postgresql.org/message-id/5537EFEF.6060808%40iki.fi), [#42 (Andrey Borodin, 2026-07-31)](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru)

**Question:** What guarantees that the ancestor-timeline filename of the switchpoint segment is in the archive before the standby recycles it?

Counter-check: held. Message #42 raises this against v8 and the thread holds no answer or archive-only recovery test. An explanation or a test in a new version can settle it.

### 5. archive_mode = always with an idempotent archive_command already covers this

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years"), wire protocol · `already-exists-1`

An operator sets archive_mode = always on primary and standby, with a shared archive and an archive_command that returns success when an identical file already exists. After the primary is lost, the archive has no gap. The only cost is more archive requests, so core would carry a second mode, protocol message and GUC to save storage requests.

Message #31 says that always plus a race-free, duplicate-tolerant archive_command makes shared unnecessary. Message #32 and message #39 do not say always fails; they say it costs 2.5 to 3 times more requests to the archive storage. That is a cost of one tool's design, which a HEAD check would reduce, as message #32 notes. The v8 fallback in message #41 also lets primary and cascading standby archive the same segments, so a duplicate-tolerant archive_command is needed anyway.

Thread: [#31 (Heikki Linnakangas, 2015-05-13)](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#32 (Andrey Borodin, 2026-02-12)](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#39 (Kirill Reshke, 2026-07-22)](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41 (Kirill Reshke, 2026-07-30)](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com)

**Question:** Given that v8 already needs a duplicate-tolerant archive_command, what can shared mode do that archive_mode = always with that command cannot do, beyond fewer archive requests?

Counter-check: held. Messages #32 and #39 state only a cost reason against always, so the question is fair. Its weight is lower because three fleet operators report the problem, and the extra requests come from always mode itself, not from one tool.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection replication-ondisk-3: Marking ancestor switchpoint segment done may leave an archive gap |
| WAL / replication / standby | **yes** | Changes when a standby marks streamed segments .ready and .done, and so when it removes WAL. |
| wire protocol | **yes** | Adds replication message 'a' (WAL archival report) sent by the walsender. |
| catalog changes | **yes** | Adds column primary_last_archived to the system view pg_stat_wal_receiver. |
| ABI / extension API | **yes** | Adds an archive_mode value in xlog.h, and external physical replication clients must handle the new message. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | Adds archive_status_report_interval and a new value shared for archive_mode. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: Unsolicited 'a' replication message breaks existing physical protocol clients |
| maintenance burden ("who maintains this in 10 years") | **yes** | WAL recycling, walsender, walreceiver and timeline handling gain a third archive mode with a fallback path. |
| performance on the hot path | **yes** | The walsender sends periodic reports and the standby scans archive_status; the thread shows no numbers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #31 says archive_mode = always with a duplicate-tolerant archive_command covers the same case. |

## 5. Questions the author must answer before anyone reviews code

1. In v8, does the walsender send 'a' only to clients that asked for it in START_REPLICATION, and what does an old pg_receivewal see?
2. Which archive_command contract does shared mode need at promotion and in the v8 fallback, and does the documented non-overwriting example meet it?
3. What caps WAL retention on a shared-mode standby during a long archive outage, and can an operator change that cap without a restart?
4. What guarantees that the ancestor-timeline filename of the switchpoint segment is in the archive before the standby marks it .done?
5. Since v8 needs a duplicate-tolerant archive_command, what does shared mode give over archive_mode = always beyond fewer archive requests?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the archiver, WAL recycling at restartpoints, timeline switches and the walsender protocol. No committer has replied since message #31.
- **Liability:** The owner signs for a new replication message, a new GUC and a third WAL recycling mode on standbys. Errors there show up as PITR gaps at restore time or as full disks on standbys, possibly years later.
- **Author follow-up:** Good sign: two authors posted v6 to v8, fixed bugs found in production use of v4, and keep the CommitFest entry current.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Operators of large fleets report PITR gaps after failover, so the goal has users. The patch touches xlog.c, walsender and walreceiver, and message #42 lists open design items, so a committer who knows the area is unlikely to start before those are settled.

## Appendix

### Held up in the counter-check, not listed above

- `maintenance-fit-4` (fixable, level 2) No committer who knows archiving and timelines is engaged. Not listed: not among the strongest objections.
- `already-exists-2` (fixable, level 1b) Expose primary_last_archived and use wal_keep_size, without a new mode. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` Fallback archiving makes shared mode depend on duplicate-tolerant archive_command. Refuted (not specific): Duplicate of prior-attempts-1, which covers both the promotion case and the v8 fallback case.
- `standards-compat-3` Meaning of archive_mode = on during recovery is not stated. Refuted (already handled): Message #41 describes an on -> on -> shared cascade as a setup v8 handles, so a standby with archive_mode = on starts in v8. No 2026 message proposes changing what on does in recovery, so the second branch has no support.
- `replication-ondisk-1` Shared mode still needs a duplicate-tolerant archive_command. Refuted (not specific): Duplicate of prior-attempts-1.
- `replication-ondisk-2` No bound on WAL a shared-mode standby retains. Refuted (not specific): Duplicate of maintenance-fit-3, which also describes the repeated failover.
- `replication-ondisk-4` Unsolicited 'a' replication message breaks existing protocol clients. Refuted (not specific): Duplicate of standards-compat-1, which also covers older pg_receivewal binaries.
- `maintenance-fit-1` v8 fallback brings back dual archiving that shared mode was meant to avoid. Refuted (not specific): Duplicate of prior-attempts-1. Its extra point about a smaller cost gap is not supported, because the fallback archives only when no reports arrive.
- `maintenance-fit-2` WAL recycling gains a third mode that future changes can miss. Refuted (already handled): Message #38 adds TAP test 056, which runs the restartpoint path where the XLogArchiveCheckDone bug removed .ready segments, and reworks the cascading tests. Message #41 registers the tests in meson.build, so a later change that breaks retention fails in CI.

### Review angles

- prior-attempts: 1 objection(s). The thread holds its own prior attempt: Heikki's 2014-2015 shared mode, cut back to always mode in message #31 and revived in message #32. The objection above covers the reason it was set aside.
- standards-compat: 3 objection(s). Objections listed; config.sgml and monitoring.sgml docs for the new GUC and column are still missing per message #42, so the full user-facing contract cannot be checked yet.
- replication-ondisk: 4 objection(s)
- security: 0 objection(s). Within the security angle I found no new exposure. The 'a' message travels only on replication connections, which already need the REPLICATION attribute. primary_last_archived matches last_archived_wal, which pg_stat_archiver already shows, and archive_command stays under superuser control. Unbounded WAL retention on a standby is an availability problem caused by an archive outage or a wrong configuration, not by an unprivileged user, so it belongs to the design angle.
- maintenance-fit: 4 objection(s). Four objections listed. The demand is real: several large fleets report it in messages #33, #37 and #39.
- already-exists: 2 objection(s). Two objections from the already-exists angle. Both rest on mechanisms the thread itself names, and neither is fatal.
- demand: 0 objection(s). The material shows real, independent demand. Borodin reports PITR gaps in 1–2% of clusters after large outages (message #32), and Kanchu (message #33) and Smolkin (message #37, tens of thousands of clusters) report the same gap and external workarounds. Heikki called the shared-archive setup very common in 2014 (message #1). The archive_mode = always workaround costs about 2.5 to 3 times the archive requests (message #39). Costs fall mostly on shared-mode users; protocol and retention risks belong to other angles.

### Precedents retrieved

- P1: [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (revert, Partially reverted, 2013-02-15) - used in the review
- P2: [Track number of WAL files ready to be archived in pg_stat_archiver](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Statistics_Collector) (do_not_want, Not worth doing, date unknown) - judged not relevant; reason summarised from its thread
- P3: [Revert "Correct WAL space calculation formula in docs."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2cae0aeb9c61aef48b6de3fb5c4f7785a06bca0b) (revert, Reverted, 2010-10-28) - judged not relevant
- P4: [Revert addition of poorly-thought-out DUMP TIMESTAMP archive entry, which induced bug #1597 in addition to having several other misbehaviors (like labeling...](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=348f856dc57902a8f951a570284918761da1776b) (revert, Reverted, 2005-04-15) - judged not relevant
- P5: [Allow archive_mode to be changed without server restart?](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Point-In-Time_Recovery_.28PITR.29) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P6: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P7: [Change walsender so that it applies per-role settings](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Continue to allow VACUUM to mark last block of index dirty even when there is no work to do.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2d2841a56c8fa37a5dd5c6d33488ba6ca37116ff) (revert, Partially reverted, 2011-11-22) - judged not relevant
- P10: [Revert part of recent include patch not ready for application.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=51e888219745cf9efa41034b5411eee7e9a8f4f1) (revert, Reverted, 2006-07-14) - judged not relevant
- P11: [Revert "Add mode where contrib installcheck runs each module in a separately named database."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fc5c1bbbeb3e00699f87407eebf2933595e9d93b) (revert, Reverted, 2012-12-03) - judged not relevant
- P12: [Revert accidental change of WAL_DEBUG default.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b29e7151434b4403d70fd35da25cec585fbd3114) (revert, Reverted, 2014-06-17) - judged not relevant
- P13: [Allow pg_xlogfile_name() to be used in recovery mode](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Standby_server_mode) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P14: [Revert "Allow units to be specified in relation option setting value."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9df492664a74a948c87232f523eea637920a516b) (revert, Reverted, 2014-08-28) - judged not relevant
- P15: [Revert WAL posix_fallocate() patches.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b1892aaeaaf34d8d1637221fc1cbda82ac3fcd71) (revert, Reverted, 2013-09-05) - judged not relevant
- P16: [Revert all keepalive moves, reevaluate.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=54aaef864213d7b91f1efc691d3dc93ba468815b) (revert, Reverted, 2010-03-13) - judged not relevant
- P17: [Revert 1fbc30f1dca6624c01a9a6a9ff711a7036c68532 because the --enable-shared is about Plyton's configure, not ours.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=39d0653d058b87b437be05fd295ceb5bb4e9e05c) (revert, Reverted, 2012-08-30) - judged not relevant
- P18: [Revert to our pre-7.4 behavior of identifying Unix-socket connections in ps status as '\[local\]', not as 'localhost' as the code has been doing recently.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b43e325ebfab423c937ce384bcdaf70ed0b462e) (revert, Reverted, 2003-09-12) - judged not relevant
- P19: [Revert wrong SCO OpenServer report, update comments and improve formatting a bit. Regenerate INSTALL.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=248365f2c64fb8ce94c28c680d95a279dd07953a) (revert, Reverted, 2001-04-06) - judged not relevant
- P20: [Revert ill-starred change of 13-Feb-02:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=303a257b5fece11ed774359b8f564ba9e694f35a) (revert, Reverted, 2004-01-14) - judged not relevant

### Warnings

- CommitFest entry has 2 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com). Others: https://www.postgresql.org/message-id/flat/EBB5D20D-8630-4557-999F-999BAD90BCCF%40yandex-team.ru
- tone lint: 2 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 206 s; cost reported by the CLI: $3.90
- Precedent enrichment: 1 thread(s) lacked a recorded reason: 1 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 20 kept
- Tone check: 121 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 2 left
  - `refuted[2].failure_scenario`: exclamation mark ("!")
  - `refuted[5].failure_scenario`: exclamation mark ("!")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:47:12.851Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
