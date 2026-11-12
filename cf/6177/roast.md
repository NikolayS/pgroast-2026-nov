# Design pre-review: Add archive_mode to prevent unarchived WAL on permanent primary failure

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6177](https://commitfest.postgresql.org/patch/6177/); [thread](https://www.postgresql.org/message-id/flat/548AF1CB.80702%40vmware.com), 42 of 42 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is stated and three fleet operators report it, so level 1 holds. At level 2, four design points are open in v8: the archive_command contract (prior-attempts-1), the 'a' message reaching clients that did not ask (standards-compat-1), unbounded WAL retention (maintenance-fit-3) and ancestor timeline marking (replication-ondisk-3). Shared mode was set aside in message #31 in favour of always mode, so the list also needs to agree on already-exists-1. All of these are fixable.

Not checked: No code was read, and the documentation hunks shown are from the 2015 patch, so v8 behaviour is known only from thread messages. The opt-in rule for 'a', the fallback timing and the timeline marking could not be checked.

## Prior attempts

- [Better fix for "unarchived WAL files get deleted on crash recovery" bug.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1bd42cd70abdbc946ad64c3c8eaefed4bb8b1145) (related; Partially reverted, 2013-02-15). "We create a .done file for files streamed or restored from archive, so the WAL file recycling logic used during normal operation works just as well during archive recovery." Different now: P1 relies on creating .done for streamed segments so that recycling in recovery works as in normal operation. Shared mode creates .ready for them instead, and the v4 XLogArchiveCheckDone bug (messages #36, #38) came from that change.

## Objections a reviewer is likely to raise

### 1. Shared mode still needs a duplicate-tolerant archive_command, the reason it was dropped

An operator follows the documented archive_command, which refuses to overwrite an existing file. The primary archives a segment and dies before its report reaches the standby, or a v8 fallback standby archives in parallel with the primary. After promotion the standby's archiver retries the same file indefinitely, .ready files pile up in pg_wal, and the operator learns of it when the disk fills.

In message #31, shared mode was set aside because always mode plus a race-free, duplicate-tolerant archive command does the job, and shared mode was kept only for its "less strict requirements for the archive_command". Messages #29 and #30 showed shared mode still needs that tolerance at promotion, and message #20 objected to adding more archive_command logic that users must get right. v8 widens the need, because its fallback (message #41) lets the primary and a cascading standby both archive. The author's stated gain over always mode is a 2.5-3x cut in archive requests (message #39), not a simpler archive_command.

**Question:** With the documented example archive_command, what happens after promotion or v8 fallback when the segment already exists in the archive, and does the patch ship a command that handles this?

<sub>fixable, level 2, prior rejection; messages [#20](https://www.postgresql.org/message-id/CA%2BTgmoY7wkYi-27LOnm128RRw8cQ%3DtNhJdPGgJ0JgpQZ-uwUuA%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/55534930.4040905%40iki.fi), [#30](https://www.postgresql.org/message-id/CA%2BTgmoY%2Bp5oowv1%3DZ44Zu14V70XKQZBkt%2BoEmwB%2BgAYYfDAV1Q%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#39](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Unsolicited 'a' replication message breaks existing physical protocol clients

An operator runs pg_receivewal, or a third-party backup tool, with a permanent slot against a primary that has archive_mode = shared. The walsender sends an 'a' message, and the client stops with "unrecognized streaming header". With the retry loop it reconnects and fails again without progress, so the WAL copy silently stalls.

Message #40 reproduced this with pg_receivewal and noted that the PG 18 'p'/'s' pair is sent only on request, to avoid this hazard. Message #41 adds a standby-side request parameter in v8 but does not say whether the walsender still sends 'a' to clients that did not ask. Fixing receivelog.c covers in-core tools only. Older pg_receivewal binaries and external clients keep failing.

**Question:** In v8, does the walsender send 'a' only to clients that asked for it in START_REPLICATION, and what does an old pg_receivewal see?

<sub>fixable, level 2, wire protocol; messages [#40](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `standards-compat-1`</sub>

### 3. No bound on WAL a shared-mode standby keeps

The archive storage goes down for a day. Every shared-mode standby keeps all streamed WAL and fills its disk. Because archive_mode is PGC_POSTMASTER, the operator cannot turn retention off without a restart, and HA fails over to the next standby, which repeats the cycle.

Message #40 (point 7) reports this from production and says downstream added a SIGHUP GUC to escape it. Message #41's v8 handles only an upstream that never reports. It does not cap retention when reports stop because archiving stalls. Replication slots needed max_slot_wal_keep_size for the same class of problem.

**Question:** What limits WAL growth on a shared-mode standby during a long archive outage, and how does an operator release it without a restart?

<sub>fixable, level 2, design; messages [#40](https://www.postgresql.org/message-id/D33EE83C-0124-48CA-B351-22B50BBD5FCD%40yandex-team.ru), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `maintenance-fit-3`</sub>

### 4. Marking ancestor switchpoint segment done may leave an archive gap

After a timeline switch, the standby marks the ancestor-timeline segment holding the switchpoint as .done because the upstream reported the descendant segment archived. If the ancestor filename was never archived, a later PITR with recovery_target_timeline set to the ancestor cannot find it.

Message #42 points out that a descendant report only proves the shared prefix exists under the descendant filename. No explanation or archive-only recovery test was given. This marks a segment as safe to recycle without proof, the gap the feature exists to prevent, and message #12 names PITR to the old timeline as a real use.

**Question:** What guarantees that the ancestor-timeline filename of the switchpoint segment is in the archive before the standby recycles it?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#12](https://www.postgresql.org/message-id/CAB7nPqS%3DV%3DLF-JZsv9eGF84joQ3jxugt4gkOK6kD1CiB98_vWg%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/5537EFEF.6060808%40iki.fi), [#42](https://www.postgresql.org/message-id/BEFEAD83-C128-4138-A03B-B5C676BE5E58%40yandex-team.ru); `replication-ondisk-3`</sub>

### 5. archive_mode = always with an idempotent archive_command already covers this

An operator sets archive_mode = always on primary and standby, with a shared archive and an archive_command that returns success when an identical file already exists. After the primary is lost, the archive has no gap. The only cost is more archive requests, so core would carry a second mode, protocol message and GUC to save storage requests.

Message #31 says that always plus a race-free, duplicate-tolerant archive_command makes shared unnecessary. Message #32 and message #39 do not say always fails; they say it costs 2.5 to 3 times more requests to the archive storage. That is a cost of one tool's design, which a HEAD check would reduce, as message #32 notes. The v8 fallback in message #41 also lets primary and cascading standby archive the same segments, so a duplicate-tolerant archive_command is needed anyway.

**Question:** Given that v8 already needs a duplicate-tolerant archive_command, what can shared mode do that archive_mode = always with that command cannot do, beyond fewer archive requests?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#31](https://www.postgresql.org/message-id/555388AF.1080103%40iki.fi), [#32](https://www.postgresql.org/message-id/D4B53AE3-B7AF-4BE6-9CB6-44956B05DE72%40yandex-team.ru), [#39](https://www.postgresql.org/message-id/CALdSSPj5CAjdn7UGzBbMJQC4SX8tvxZ8%2B2bVU2KYngY9%2BHTvUg%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/CALdSSPiVbHM9%2BXettYcOHDJ6KTLzjNEhdURwXwfzyMV-dtL1Gg%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. In v8, does the walsender send 'a' only to clients that asked for it in START_REPLICATION, and what does an old pg_receivewal see?
2. Which archive_command contract does shared mode need at promotion and in the v8 fallback, and does the documented non-overwriting example meet it?
3. What caps WAL retention on a shared-mode standby during a long archive outage, and can an operator change that cap without a restart?
4. What guarantees that the ancestor-timeline filename of the switchpoint segment is in the archive before the standby marks it .done?
5. Since v8 needs a duplicate-tolerant archive_command, what does shared mode give over archive_mode = always beyond fewer archive requests?

## Touch points

- **on-disk format / pg_upgrade:** see objection replication-ondisk-3: Marking ancestor switchpoint segment done may leave an archive gap
- **WAL / replication / standby:** Changes when a standby marks streamed segments .ready and .done, and so when it removes WAL.
- **wire protocol:** Adds replication message 'a' (WAL archival report) sent by the walsender.
- **catalog changes:** Adds column primary_last_archived to the system view pg_stat_wal_receiver.
- **ABI / extension API:** Adds an archive_mode value in xlog.h, and external physical replication clients must handle the new message.
- **new GUC:** Adds archive_status_report_interval and a new value shared for archive_mode.
- **backpatchability:** see objection standards-compat-1: Unsolicited 'a' replication message breaks existing physical protocol clients
- **maintenance burden ("who maintains this in 10 years"):** WAL recycling, walsender, walreceiver and timeline handling gain a third archive mode with a fallback path.
- **performance on the hot path:** The walsender sends periodic reports and the standby scans archive_status; the thread shows no numbers.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #31 says archive_mode = always with a duplicate-tolerant archive_command covers the same case.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the archiver, WAL recycling at restartpoints, timeline switches and the walsender protocol. No committer has replied since message #31.
- Priority: fix design first; audience a sizeable minority; review cost high. Operators of large fleets report PITR gaps after failover, so the goal has users. The patch touches xlog.c, walsender and walreceiver, and message #42 lists open design items, so a committer who knows the area is unlikely to start before those are settled.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
