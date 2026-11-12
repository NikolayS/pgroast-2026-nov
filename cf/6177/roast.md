# Design pre-review: Add archive_mode to prevent unarchived WAL on permanent primary failure

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6177](https://commitfest.postgresql.org/patch/6177/); [thread](https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com), 42 of 42 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Several operators report the PITR gap, so the goal is not in doubt. The design has open safety and retention points: replication-ondisk-3 (ancestor-timeline segments marked .done without proof), replication-ondisk-1 (no bound on standby WAL retention) and prior-attempts-1 (no archive_command contract for duplicates). already-exists-1 and already-exists-2 ask what shared mode saves over always mode or a smaller first step. The list needs to settle these before code review.

Not checked: The v8 code was not read. Whether 'a' is sent only on request, how the fallback timer works, and how shared mode behaves when the standby restores from the archive could not be checked.

## Prior attempts

- [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (related; Partially reverted, 2013-02-15). "We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery." Different now: P4 set the rule that streamed and restored segments get .done so normal recycling works in archive recovery. Shared mode reverses this for streamed segments and keeps them .ready, which needs its own recycling guarantees.

## Objections a reviewer is likely to raise

### 1. Ancestor-timeline segments marked .done without proof they are archived

After a timeline switch the standby receives a report for the descendant timeline. It marks the ancestor's switchpoint segment .done and recycles it, but the archive holds only the descendant-named file. A later PITR with recovery_target_timeline set to the ancestor cannot find that segment, and the gap stays hidden until a restore is attempted.

Message #42 raises this as open: a report on a descendant timeline proves only the common prefix under the descendant filename. The feature exists to remove PITR gaps, so marking a file .done that may be absent from the archive breaks the invariant the mode promises. Messages #21 to #25 show that the switchpoint segment naming is subtle and that recovery depends on it.

**Question:** What guarantees that the ancestor-named switchpoint segment is in the archive before the standby marks it .done, and which archive-only recovery test shows it?

<sub>fixable, level 2, WAL / replication / standby; messages [#21](https://www.postgresql.org/message-id/5537EFEF.6060808%40iki.fi), [#25](https://www.postgresql.org/message-id/5538A927.7000407%40iki.fi), [#42](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru); `replication-ondisk-3`</sub>

### 2. Standby WAL retention has no bound when archive reports stop

The archive storage goes down for a day while the primary keeps writing WAL. Every shared-mode standby keeps all streamed segments as .ready and fills pg_wal, then stops. After failover the next standby is in the same state, so the whole cluster loses availability because of a backup outage.

Message #40 reports this from production and says the downstream fork needed a SIGHUP escape hatch, because archive_mode is PGC_POSTMASTER. v8 (message #41) handles an upstream that is configured not to archive, through the FATAL and the fallback. It does not handle an upstream that is configured to archive but cannot. No cap similar to max_slot_wal_keep_size is proposed, and the docs do not describe this failure mode.

**Question:** What limits pg_wal growth on a shared-mode standby while archive_command fails on the primary, and how does an operator release that WAL without a restart?

<sub>fixable, level 2, WAL / replication / standby; messages [#40](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. Duplicate archival, which stalled shared mode in 2015, is still unresolved

In a v8 setup the standby runs fallback archiving after a quiet period while the primary archives the same segment. An archive_command that refuses to overwrite, as the manual recommends, returns failure. The archiver then retries that segment forever and WAL piles up in pg_wal until someone notices the disk filling.

In message #29 Heikki confirmed that a promoted shared-mode standby gets stuck retrying segments the primary had already archived. In message #31 he cut the patch back to always, because the fix needs a duplicate-tolerant command. Message #20 asked the project not to add more cases that users must "get right" in archive_command. The v8 fallback (message #41) makes double archival routine, and message #42 shows its timing can race the primary, yet the proposal does not yet define the archive_command contract for identical files.

**Question:** What exact archive_command behavior does shared mode require for a segment already archived, and how does the archiver avoid looping on it with the documented example command?

<sub>fixable, level 2, prior rejection; messages [#20](https://www.postgresql.org/message-id/CA%2BTgmoY7wkYi-27LOnm128RRw8cQ%3DtNhJdPGgJ0JgpQZ-uwUuA%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#30](https://www.postgresql.org/message-id/CA%2BTgmoY%2Bp5oowv1%3DZ44Zu14V70XKQZBkt%2BoEmwB%2BgAYYfDAV1Q%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru); `prior-attempts-1`</sub>

### 4. archive_mode = always with a duplicate-tolerant archive_command already closes the gap

An operator who runs archive_mode = always on standbys with a shared archive and a duplicate-tolerant archive_command already gets a gap-free archive after promotion. Shared mode adds a second archiving mode, a replication message, a GUC and fallback logic for the same outcome. Every later change to archiving or walsender then has two cooperating paths to keep correct.

In message #31 Heikki dropped shared mode, saying always plus a race-free archive_command that handles duplicates makes it unnecessary. The only answer is cost: about 2.5 to 3 times more archive storage requests in WAL-G (messages #32, #39). That makes always more expensive, not impossible. Shared mode does not remove the duplicate requirement either: promotion re-archives what the primary may already have stored (message #29), and v8 fallback lets primary and standby archive the same segment (message #41).

**Question:** Since shared mode still needs a duplicate-tolerant archive_command for promotion and fallback, which setups cannot use archive_mode = always, beyond the extra storage requests?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#29](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#31](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#32](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#39](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `already-exists-1`</sub>

### 5. Exposing the upstream archive position may be enough for external tools

Core takes on WAL retention, cascading relay, request negotiation, a FATAL on connect and timed fallback archiving. The open v8 issues show these are hard to get right: idle-period races, wal_sender_timeout = 0, ancestor timelines. HA tools would have managed with a queryable primary_last_archived and a bounded retention setting.

In message #33 a second operator says that exposing last_archived_wal on the standby alone "would unblock everyone solving this problem externally". That is the 0002 piece. Standby-side retention does need core help, and a committer would likely ask whether that help can stay small. One candidate is a primary_last_archived column plus a retention rule, with promotion-time archiving left to the HA tool, which already re-archives from the standby (message #32).

**Question:** What would HA tools still be unable to do if core only exposed primary_last_archived and held back recycling of unconfirmed segments, without fallback archiving?

<sub>fixable, level 1b, desirability; messages [#32](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#33](https://www.postgresql.org/message-id/41D301A5-AB81-4B21-8AC1-9DD602362D31%40apple.com), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru); `already-exists-2`</sub>

## Questions for the author

1. Since promotion and the v8 fallback still need a duplicate-tolerant archive_command, which setups cannot use archive_mode = always, beyond the extra storage requests?
2. What guarantees that the ancestor-named switchpoint segment is in the archive before the standby marks it .done, and which archive-only recovery test shows it?
3. What limits pg_wal growth on a shared-mode standby while archive_command fails on the primary, and how does an operator release that WAL without a restart?
4. What must archive_command do with a segment that is already in the archive, and how does the documented no-overwrite example avoid an endless retry?
5. In v8, does any walsender send 'a' to a client that did not request it, and how does an older pg_receivewal behave against a shared-mode primary?

## Touch points

- **on-disk format / pg_upgrade:** Shared mode changes when the standby creates .ready and .done files and when it recycles segments in pg_wal.
- **WAL / replication / standby:** The standby keeps streamed WAL until the upstream reports it archived, and cascading standbys relay the report.
- **wire protocol:** The patch adds walsender message Byte1('a') and, in v8, a request for it from the standby.
- **catalog changes:** The patch adds column primary_last_archived to the pg_stat_wal_receiver view.
- **new GUC:** The patch adds archive_status_report_interval and a new archive_mode value, shared.
- **maintenance burden ("who maintains this in 10 years"):** The mode spans archiver, walsender, walreceiver and recovery, with relay, negotiation, FATAL and timed fallback states.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** archive_mode = always or an exposed primary_last_archived with external tools may cover the same need.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the archiver, walsender and walreceiver, and timeline handling in recovery. The 2014 design came from a committer, but none has commented on v6 to v8.
- Priority: fix design first; audience a sizeable minority; review cost high. Several operators with shared archives report PITR gaps, so the goal has users. The patch adds a protocol message, a GUC and new recovery semantics, and the co-author's own review of v8 lists open design issues that should close before a committer spends time.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 8 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
