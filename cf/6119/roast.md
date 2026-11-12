# Design pre-review: Migrate pg_commit_ts directory to the new node during an upgrade

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6119](https://commitfest.postgresql.org/patch/6119/); [thread](https://www.postgresql.org/message-id/flat/182311743703924%40mail.yandex.ru), 60 of 60 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support, but the design depends on replication origin preservation that the thread has not settled. replication-ondisk-1 shows that the copied entries point at reassigned roident values when 0003 is dropped and patch 6743 is not committed. standards-compat-2 and standards-compat-1 show that the lasting meaning of --pg-commit-ts and the scope of the new hard error are undecided. These are fixable, but the list must agree on them before code review is useful.

Not checked: Only the first 3 of 60 messages and no v16 code were read. This review could not check how the copy sets the commit-ts XID range, how it treats existing entries in the new cluster, or how it handles old clusters before 9.5.

## Prior attempts

None found before 2025-04-03.

## Objections a reviewer is likely to raise

### 1. Copied commit timestamps carry roident values the new cluster does not preserve

An admin upgrades a logical subscriber with --pg-commit-ts. Subscriptions get new roident values, so rows applied by one subscription are attributed to another, and pg_replication_origin_create() origins are not migrated at all. The apply worker logs false update_origin_differs conflicts, and audit queries on pg_xact_commit_timestamp_origin() return wrong origins.

Each commit_ts entry stores a roident, and message #37 shows that pg_upgrade reassigns roident values. 0002 makes the copy opt-in, but the copied data is still wrong for anyone who opts in while 0003 is dropped, as message #60 requests. Messages #59 and #60 tie safe use to patch 6743, which is not part of this proposal. Message #44 says manually created origins are not migrated.

**Question:** With 0003 dropped, what keeps the copied roident values pointing at the right origin, including origins made with pg_replication_origin_create()?

<sub>fixable, level 2, WAL / replication / standby; messages [#37](https://www.postgresql.org/message-id/CAD21AoDG8zQpHHfw7OvaEy7W0ZSyP%3D_dS-hrcquJ3C_ctMDmMQ%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/CAA4eK1JzhWpyni%3DtaOwsH%3DwDMgnT%2Bact0NddSOGJgEBs0YZOUA%40mail.gmail.com), [#44](https://www.postgresql.org/message-id/6741774083455%40mail.yandex.ru), [#47](https://www.postgresql.org/message-id/5007211775411625%40mail.yandex.ru), [#56](https://www.postgresql.org/message-id/OS9PR01MB1214953D981C1DFC9B29481EAF5AF2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#59](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com), [#60](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Option described as temporary becomes a permanent command-line contract

Operators add --pg-commit-ts to their upgrade scripts. Once origin preservation lands, the project either keeps an option with no remaining purpose or removes it. If it is removed, those scripts fail on the next major version with an unknown-option error.

Message #46 calls the option a temporary solution to remove once an architecturally correct design exists. Message #57 says its side effects go away after patch 6743. Every pg_upgrade option in a release must be supported for later releases, so the long-term meaning of this option needs to be settled before it is added.

**Question:** Is --pg-commit-ts meant to stay once origins are preserved, and if so, what does it mean then?

<sub>fixable, level 2, design; messages [#46](https://www.postgresql.org/message-id/53871774622072%40mail.yandex.ru), [#56](https://www.postgresql.org/message-id/OS9PR01MB1214953D981C1DFC9B29481EAF5AF2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#57](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru), [#59](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com); `standards-compat-2`</sub>

### 3. New hard error breaks existing upgrade scripts when commit timestamps are off

A site with track_commit_timestamp = on in the old cluster leaves it off in the new cluster, as it can today. After moving to the new pg_upgrade, the check step fails with "track_commit_timestamp" must be "on", even though the site never asked for timestamps to be copied. The site finds out during the upgrade window.

Message #11 added the check before the opt-in option existed, and message #37 asked for a warning instead, since copying is not mandatory. Message #44 then made copying opt-in. The brief says v16 does not state whether the check still fires without --pg-commit-ts. If it does, pg_upgrade rejects a setup it accepts today.

**Question:** Does the track_commit_timestamp check fire only when --pg-commit-ts is given, and what happens otherwise?

<sub>fixable, level 2, design; messages [#7](https://www.postgresql.org/message-id/CAA4eK1%2BZayox%3Dx84upet_gc4hrCKSwZSpnO9%3DQ4vH8rrJbsBOg%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/56771759769555%40mail.yandex.ru), [#37](https://www.postgresql.org/message-id/CAD21AoDG8zQpHHfw7OvaEy7W0ZSyP%3D_dS-hrcquJ3C_ctMDmMQ%40mail.gmail.com), [#44](https://www.postgresql.org/message-id/6741774083455%40mail.yandex.ru), [#57](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru); `standards-compat-1`</sub>

### 4. Option name describes a directory, not the behaviour users get

A user who wants commit timestamps after an upgrade searches the pg_upgrade options for "timestamp" or "preserve" and does not find --pg-commit-ts. The user upgrades without the option, and the timestamps are lost with no way to get them back.

Message #60 notes that the name and help text describe the implementation and proposes --preserve-commit-timestamps. The thread does not settle this. Message #60 also says the help line breaks the alphabetical order of the long options.

**Question:** Which name do pg_upgrade users understand without reading the source, and does the help text say what is preserved?

<sub>fixable, level 2, design; messages [#60](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com); `standards-compat-3`</sub>

### 5. Without origin preservation, the shippable part serves mainly one site's audit use

0001 and 0002 are committed and patch 6743 is not. Logical replication users leave --pg-commit-ts off to avoid false update_origin_differs conflicts, so the conflict-detection use case gains nothing. The new option then serves mainly sites that use commit timestamps as a change-time history, like the author's.

The only field report is the author's site, which keeps commit times for up to two years to find application errors (message #42). Kuroda asked for a stronger use case (message #41), and Sawada's non-replication example is hypothetical (message #39). Amit asked what value the patch has without origins (message #38), and messages #57, #59 and #60 agree the option exists because origins are not yet preserved. Users who need the history can copy the directory and run pg_resetwal today (message #2), and the author calls that unreliable but still uses it.

**Question:** Apart from your own site, which users rely on commit timestamps surviving an upgrade without logical replication, and is that enough to justify this before origins are preserved?

<sub>fixable, level 1b, demand; messages [#2](https://www.postgresql.org/message-id/CACG%3Deza020LBv6-PPHFTi8OET1971bi8on98Uw6AtW7yV%3DeX%2Bg%40mail.gmail.com), [#38](https://www.postgresql.org/message-id/CAA4eK1JzhWpyni%3DtaOwsH%3DwDMgnT%2Bact0NddSOGJgEBs0YZOUA%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CAD21AoDjAbmzUTmdB8iY3pKYVA6qAMkgYyBeeDm3WDyqVU4X%2BQ%40mail.gmail.com), [#41](https://www.postgresql.org/message-id/OS9PR01MB12149BC0A686B802D4C8D4FE3F54FA%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#42](https://www.postgresql.org/message-id/524651773902207%40mail.yandex.ru), [#57](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru), [#59](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com), [#60](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. With 0003 dropped and patch 6743 not yet committed, what keeps copied roident values pointing at the right origin, including origins made with pg_replication_origin_create()?
2. Should this patch wait for patch 6743, or is it meant to be committed before origin preservation exists?
3. Once origins are preserved, does --pg-commit-ts stay, and what does it mean then?
4. Does the track_commit_timestamp check fire only when --pg-commit-ts is given, and what happens to existing upgrades otherwise?
5. What does pg_upgrade do with commit-ts entries that already exist in the new cluster before the copy?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade copies the pg_commit_ts SLRU files and sets the commit-ts XID range with pg_resetwal -c.
- **WAL / replication / standby:** Copied entries carry roident values that logical replication conflict detection reads on an upgraded subscriber.
- **catalog changes:** 0003 restores roident and remote_lsn of replication origins in the new cluster.
- **backpatchability:** see objection standards-compat-1: New hard error breaks existing upgrade scripts when commit timestamps are off
- **maintenance burden ("who maintains this in 10 years"):** A new pg_upgrade option, described in message #46 as temporary, and a new check step must be supported in later releases.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: Without origin preservation, the shippable part serves mainly one site's audit use

## Who would commit it, and when

- Owner: qualified committer engaged. The review notes record committer support for the logical replication use case in messages #27 and #38. The owner would need to know pg_upgrade and replication origins.
- Priority: fix design first; audience niche; review cost medium. The feature helps sites that run with track_commit_timestamp = on, mainly logical replication subscribers and audit users like the author's site. The patch is small, but its correctness depends on the open origin preservation question and on the lasting meaning of the new option.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
