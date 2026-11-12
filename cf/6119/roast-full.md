# Design pre-review: Migrate pg_commit_ts directory to the new node during an upgrade

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6119](https://commitfest.postgresql.org/patch/6119/) (Needs review); authors: Sergey Levin (lsv)
- Thread: [Patch for migration of the pg_commit_ts directory](https://www.postgresql.org/message-id/flat/182311743703924%40mail.yandex.ru), started 2025-04-03
- Messages read: 60 of 60
- Patch prose read: v1-0001-Migration-of-the-pg_commit_ts-directory.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support, but the design depends on replication origin preservation that the thread has not settled. replication-ondisk-1 shows that the copied entries point at reassigned roident values when 0003 is dropped and patch 6743 is not committed. standards-compat-2 and standards-compat-1 show that the lasting meaning of --pg-commit-ts and the scope of the new hard error are undecided. These are fixable, but the list must agree on them before code review is useful.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only the first 3 of 60 messages and no v16 code were read. This review could not check how the copy sets the commit-ts XID range, how it treats existing entries in the new cluster, or how it handles old clusters before 9.5.

## The proposal, restated

**Problem.** pg_upgrade does not copy the pg_commit_ts directory, so commit timestamps and their origin data are lost when track_commit_timestamp = on. The author's site uses commit times to find when rows changed, kept for up to 2 years, and copies the directory by hand today. Reviewers add a second use case: accurate logical replication conflict detection on an upgraded subscriber.

**Proposed design.** pg_upgrade copies pg_commit_ts from the old cluster and passes the old oldest and newest commit-ts XIDs to pg_resetwal -c, where it used to pass next XID twice. In v16 the work is split: 0001 copies the directory, 0002 makes this opt-in with a new option, and 0003 saves and restores roident and remote_lsn of replication origins. A check fails the upgrade if the old cluster has commit-ts data but the new cluster has track_commit_timestamp off; v16 does not say whether this check stays once the option exists.

**User-visible surface.**
- New pg_upgrade option --pg-commit-ts that turns on the copy; a reviewer proposed renaming it --preserve-commit-timestamps.
- New pg_upgrade check step that fails with "track_commit_timestamp" must be "on" but is set to "off" on the new cluster.
- Commit timestamps and origins, read by pg_xact_commit_timestamp_origin(), now survive pg_upgrade, which changes the documented note that they are not preserved.
- 0003 keeps replication origin roident values from the old cluster.

**What the author claims.**
- The only cost is the time to copy the pg_commit_ts directory.
- The patch does automatically what users do today by copying the directory and running pg_resetwal.
- The author's site needs commit times for up to 2 years to find application errors.
- Without the opt-in option, upgraded subscribers would log many update_origin_differs conflicts that did not appear before.
- The option lets users who copy the directory with their own scripts keep doing so.

**Thread state.** Kuroda now maintains a three-part v16. Maxim Orlov and Ajin Cherian say 0001 must not go in without 0002 or origin preservation, and Ajin asks to drop 0003 in favour of patch 6743, rename the option, and rebase.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-04-03; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Copied commit timestamps carry roident values the new cluster does not preserve

fixable · level 2, rubric item 7 · WAL / replication / standby, catalog changes, on-disk format / pg_upgrade · `replication-ondisk-1`

An admin upgrades a logical subscriber with --pg-commit-ts. Subscriptions get new roident values, so rows applied by one subscription are attributed to another, and pg_replication_origin_create() origins are not migrated at all. The apply worker logs false update_origin_differs conflicts, and audit queries on pg_xact_commit_timestamp_origin() return wrong origins.

Each commit_ts entry stores a roident, and message #37 shows that pg_upgrade reassigns roident values. 0002 makes the copy opt-in, but the copied data is still wrong for anyone who opts in while 0003 is dropped, as message #60 requests. Messages #59 and #60 tie safe use to patch 6743, which is not part of this proposal. Message #44 says manually created origins are not migrated.

Thread: [#37 (Masahiko Sawada, 2026-03-16)](https://www.postgresql.org/message-id/CAD21AoDG8zQpHHfw7OvaEy7W0ZSyP%3D_dS-hrcquJ3C_ctMDmMQ%40mail.gmail.com), [#38 (Amit Kapila, 2026-03-17)](https://www.postgresql.org/message-id/CAA4eK1JzhWpyni%3DtaOwsH%3DwDMgnT%2Bact0NddSOGJgEBs0YZOUA%40mail.gmail.com), [#44 (ls7777, 2026-03-21)](https://www.postgresql.org/message-id/6741774083455%40mail.yandex.ru), [#47 (ls7777, 2026-04-05)](https://www.postgresql.org/message-id/5007211775411625%40mail.yandex.ru), [#56 ("Hayato Kuroda (Fujitsu)", 2026-08-25)](https://www.postgresql.org/message-id/OS9PR01MB1214953D981C1DFC9B29481EAF5AF2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#59 (Maxim Orlov, 2026-10-16)](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com), [#60 (Ajin Cherian, 2026-11-10)](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com)

**Question:** With 0003 dropped, what keeps the copied roident values pointing at the right origin, including origins made with pg_replication_origin_create()?

Counter-check: held. Message #37 reproduces the wrong attribution, and messages #59 and #60 agree the copy is safe for subscribers only with origin preservation. The opt-in option limits who is affected but does not make the copied roident values correct.

### 2. Option described as temporary becomes a permanent command-line contract

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

Operators add --pg-commit-ts to their upgrade scripts. Once origin preservation lands, the project either keeps an option with no remaining purpose or removes it. If it is removed, those scripts fail on the next major version with an unknown-option error.

Message #46 calls the option a temporary solution to remove once an architecturally correct design exists. Message #57 says its side effects go away after patch 6743. Every pg_upgrade option in a release must be supported for later releases, so the long-term meaning of this option needs to be settled before it is added.

Thread: [#46 (ls7777, 2026-03-27)](https://www.postgresql.org/message-id/53871774622072%40mail.yandex.ru), [#56 ("Hayato Kuroda (Fujitsu)", 2026-08-25)](https://www.postgresql.org/message-id/OS9PR01MB1214953D981C1DFC9B29481EAF5AF2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#57 (Sergey, 2026-08-26)](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru), [#59 (Maxim Orlov, 2026-10-16)](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com)

**Question:** Is --pg-commit-ts meant to stay once origins are preserved, and if so, what does it mean then?

Counter-check: held. Message #46 calls the option a temporary solution, and message #56 doubts it is needed. Message #57 names a lasting use, but the thread does not settle what the option means once origins are preserved.

### 3. New hard error breaks existing upgrade scripts when commit timestamps are off

fixable · level 2, rubric item 12 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A site with track_commit_timestamp = on in the old cluster leaves it off in the new cluster, as it can today. After moving to the new pg_upgrade, the check step fails with "track_commit_timestamp" must be "on", even though the site never asked for timestamps to be copied. The site finds out during the upgrade window.

Message #11 added the check before the opt-in option existed, and message #37 asked for a warning instead, since copying is not mandatory. Message #44 then made copying opt-in. The brief says v16 does not state whether the check still fires without --pg-commit-ts. If it does, pg_upgrade rejects a setup it accepts today.

Thread: [#7 (Amit Kapila, 2025-10-02)](https://www.postgresql.org/message-id/CAA4eK1%2BZayox%3Dx84upet_gc4hrCKSwZSpnO9%3DQ4vH8rrJbsBOg%40mail.gmail.com), [#11 (ls7777, 2025-10-06)](https://www.postgresql.org/message-id/56771759769555%40mail.yandex.ru), [#37 (Masahiko Sawada, 2026-03-16)](https://www.postgresql.org/message-id/CAD21AoDG8zQpHHfw7OvaEy7W0ZSyP%3D_dS-hrcquJ3C_ctMDmMQ%40mail.gmail.com), [#44 (ls7777, 2026-03-21)](https://www.postgresql.org/message-id/6741774083455%40mail.yandex.ru), [#57 (Sergey, 2026-08-26)](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru)

**Question:** Does the track_commit_timestamp check fire only when --pg-commit-ts is given, and what happens otherwise?

Counter-check: held. Message #57 suggests the author wants track_commit_timestamp = off to stay possible in the new cluster, but nothing in v16 says the check is skipped without --pg-commit-ts. The question is open and a new patch version can answer it.

### 4. Option name describes a directory, not the behaviour users get

fixable · level 2, rubric item 10 · design · `standards-compat-3`

A user who wants commit timestamps after an upgrade searches the pg_upgrade options for "timestamp" or "preserve" and does not find --pg-commit-ts. The user upgrades without the option, and the timestamps are lost with no way to get them back.

Message #60 notes that the name and help text describe the implementation and proposes --preserve-commit-timestamps. The thread does not settle this. Message #60 also says the help line breaks the alphabetical order of the long options.

Thread: [#60 (Ajin Cherian, 2026-11-10)](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com)

**Question:** Which name do pg_upgrade users understand without reading the source, and does the help text say what is preserved?

Counter-check: held. Message #60 raises the name and the thread has no reply yet. The help text "copy pg_commit_ts directory" does not say what is preserved, so the user interface concern stands; the alphabetical order point is polish and adds no weight.

### 5. Without origin preservation, the shippable part serves mainly one site's audit use

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

0001 and 0002 are committed and patch 6743 is not. Logical replication users leave --pg-commit-ts off to avoid false update_origin_differs conflicts, so the conflict-detection use case gains nothing. The new option then serves mainly sites that use commit timestamps as a change-time history, like the author's.

The only field report is the author's site, which keeps commit times for up to two years to find application errors (message #42). Kuroda asked for a stronger use case (message #41), and Sawada's non-replication example is hypothetical (message #39). Amit asked what value the patch has without origins (message #38), and messages #57, #59 and #60 agree the option exists because origins are not yet preserved. Users who need the history can copy the directory and run pg_resetwal today (message #2), and the author calls that unreliable but still uses it.

Thread: [#2 (Maxim Orlov, 2025-09-30)](https://www.postgresql.org/message-id/CACG%3Deza020LBv6-PPHFTi8OET1971bi8on98Uw6AtW7yV%3DeX%2Bg%40mail.gmail.com), [#38 (Amit Kapila, 2026-03-17)](https://www.postgresql.org/message-id/CAA4eK1JzhWpyni%3DtaOwsH%3DwDMgnT%2Bact0NddSOGJgEBs0YZOUA%40mail.gmail.com), [#39 (Masahiko Sawada, 2026-03-17)](https://www.postgresql.org/message-id/CAD21AoDjAbmzUTmdB8iY3pKYVA6qAMkgYyBeeDm3WDyqVU4X%2BQ%40mail.gmail.com), [#41 ("Hayato Kuroda (Fujitsu)", 2026-03-19)](https://www.postgresql.org/message-id/OS9PR01MB12149BC0A686B802D4C8D4FE3F54FA%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#42 (ls7777, 2026-03-19)](https://www.postgresql.org/message-id/524651773902207%40mail.yandex.ru), [#57 (Sergey, 2026-08-26)](https://www.postgresql.org/message-id/b5af88f0-81cf-43d6-9e67-584b2b15cfc1%40yandex.ru), [#59 (Maxim Orlov, 2026-10-16)](https://www.postgresql.org/message-id/CACG%3Dezbm6RE%3DOqN9dZRb4iJoAJ%2BOPGRPQg%3D7JVdH6MZdgFwRAw%40mail.gmail.com), [#60 (Ajin Cherian, 2026-11-10)](https://www.postgresql.org/message-id/CAFPTHDbQi_dvgNhpx1QQKoEJKg7cxFxoWNfzM7DPBQ3LqDzz1Q%40mail.gmail.com)

**Question:** Apart from your own site, which users rely on commit timestamps surviving an upgrade without logical replication, and is that enough to justify this before origins are preserved?

Counter-check: held. One claim is too strong: message #30 shows conflict detection for locally changed rows works after the copy without origin preservation. The only field report is still the author's site (message #42), and message #41 asked for a stronger use case.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_upgrade copies the pg_commit_ts SLRU files and sets the commit-ts XID range with pg_resetwal -c. |
| WAL / replication / standby | **yes** | Copied entries carry roident values that logical replication conflict detection reads on an upgraded subscriber. |
| wire protocol | no |  |
| catalog changes | **yes** | 0003 restores roident and remote_lsn of replication origins in the new cluster. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: New hard error breaks existing upgrade scripts when commit timestamps are off |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new pg_upgrade option, described in message #46 as temporary, and a new check step must be supported in later releases. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: Without origin preservation, the shippable part serves mainly one site's audit use |

## 5. Questions the author must answer before anyone reviews code

1. With 0003 dropped and patch 6743 not yet committed, what keeps copied roident values pointing at the right origin, including origins made with pg_replication_origin_create()?
2. Should this patch wait for patch 6743, or is it meant to be committed before origin preservation exists?
3. Once origins are preserved, does --pg-commit-ts stay, and what does it mean then?
4. Does the track_commit_timestamp check fire only when --pg-commit-ts is given, and what happens to existing upgrades otherwise?
5. What does pg_upgrade do with commit-ts entries that already exist in the new cluster before the copy?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The review notes record committer support for the logical replication use case in messages #27 and #38. The owner would need to know pg_upgrade and replication origins.
- **Liability:** The owner signs for a third SLRU that pg_upgrade copies across major versions, for a pg_upgrade option that scripts will depend on, and for correct origin attribution on upgraded subscribers. None of these has an end date.
- **Author follow-up:** The original author still takes part late in the thread (messages #42, #46, #57), and a reviewer now maintains the v16 patch set. The CommitFest entry needs a rebase as of 2026-11-10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The feature helps sites that run with track_commit_timestamp = on, mainly logical replication subscribers and audit users like the author's site. The patch is small, but its correctness depends on the open origin preservation question and on the lasting meaning of the new option.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Raw copy freezes the commit_ts SLRU layout across major versions. Refuted (no failure scenario): The scenario depends on a future layout change that no one has proposed. pg_upgrade already copies pg_xact and pg_multixact under the same duty, so the proposal follows existing practice.
- `maintenance-fit-1` A temporary pg_upgrade option would become permanent. Refuted (not specific): Duplicate of standards-compat-2. Its extra points about the check and the option name are covered by standards-compat-1 and standards-compat-3.
- `maintenance-fit-2` Commit-ts copy is correct only after a separate patch lands. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same stale roident problem and also covers origins made with pg_replication_origin_create().

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P7] and [P17] touch pg_upgrade's handling of control data and pg_resetwal, but neither recorded reason applies: this patch sets the commit-ts XID range explicitly and reads existing pg_controldata lines without changing any output. No message in the thread cites an earlier attempt; message #2 says it is not known why pg_upgrade never copied pg_commit_ts, and patch 6743 is a parallel proposal, not a rejection.
- standards-compat: 3 objection(s)
- replication-ondisk: 2 objection(s). Kuroda's question about overwriting commit_ts entries in the new cluster seems minor: pg_upgrade already replaces pg_xact wholesale, and post-restore XIDs fall outside the copied range. The standby rsync procedure was not checked.
- security: 0 objection(s). This proposal has no new privilege or exposure aspect. pg_upgrade runs as the cluster owner, and the commit timestamps it keeps were already readable through pg_xact_commit_timestamp_origin() in the old cluster, so no role gains access it did not have before. The SQL that 0003 generates from old-cluster origin data was rejected in message #45, and message #60 asks to drop 0003, so that concern is already settled in the thread.
- maintenance-fit: 2 objection(s). The hot-path cost is limited to pg_upgrade run time, and only when the option is given. No measured copy time is shown. The listed precedents touch pg_upgrade and pg_resetwal, but their recorded reasons do not apply here.
- already-exists: 0 objection(s). The only existing alternative is the undocumented manual idiom from message #2: copy pg_commit_ts and run pg_resetwal -c with XIDs read from pg_controldata, which the author calls unreliable in message #42. Origin preservation for subscribers has no supported manual path, and the overlap of 0003 with patch 6743 is already being handled in the thread, so nothing is raised from this angle.
- demand: 1 objection(s). The logical replication use case has committer support (messages #27, #38), but it depends on origin preservation from patch 6743. For that case alone, demand is not in doubt.

### Precedents retrieved

- P1: [add timing information to pg_upgrade](https://commitfest.postgresql.org/patch/4464/) (cf, Withdrawn, 2023-12-02) - judged not relevant; reason summarised from its thread
- P2: [pg_upgrade: remove C comment](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=84a6f63e32dbefe3dc76cbe628fab6cbfc26141e) (revert, Partially reverted, 2018-01-05) - judged not relevant
- P3: [pg_upgrade documentation improvement](https://commitfest.postgresql.org/patch/583/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P4: [pg_upgrade version and path checking](https://commitfest.postgresql.org/patch/2095/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [Revert "Add basic TAP test setup for pg_upgrade"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=58ffe141eb37c3f027acd25c1fc6b36513bf9380) (revert, Reverted, 2017-09-22) - judged not relevant
- P6: [Revert "In the pg_upgrade test suite, don't write to src/test/regress."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ae35e1c9d74afc3b1e7572ad60781946fdf71e95) (revert, Reverted, 2019-05-19) - judged not relevant
- P7: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - used in the review
- P8: [pg_upgrade: Pass -j option down to vacuumdb](https://commitfest.postgresql.org/patch/1943/) (cf, Returned with feedback, 2019-04-09) - judged not relevant; reason summarised from its thread
- P9: [Preserve versions of initdb-created collations in pg_upgrade](https://commitfest.postgresql.org/patch/2328/) (cf, Returned with feedback, 2020-01-24) - judged not relevant; reason summarised from its thread
- P10: [Backport perl tests for pg_upgrade](https://commitfest.postgresql.org/patch/3997/) (cf, Withdrawn, 2022-12-19) - judged not relevant; reason summarised from its thread
- P11: [Revert "pg_upgrade: Fix quoting of some arguments in pg_ctl command"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e818866e57aa9ec13821fd36a2ff60e00e6dea6) (revert, Reverted, 2020-02-10) - judged not relevant
- P12: [Improve documentation for pg_upgrade, standbys and rsync](https://commitfest.postgresql.org/patch/3125/) (cf, Withdrawn, 2022-04-08) - judged not relevant; reason summarised from its thread
- P13: [pg_upgrade: revert part of patch for ease of translation](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3e6f01fd7d9b01b17626a6bc38cf664354eede71) (revert, Partially reverted, 2018-01-05) - judged not relevant
- P14: [Rewriting the test of pg_upgrade as a TAP test](https://commitfest.postgresql.org/patch/1101/) (cf, Returned with feedback, 2017-09-29) - judged not relevant; reason summarised from its thread
- P15: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P16: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P17: [Revert ill-considered change in pg_resetwal output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=918de37652138da7f7e54c79351178367d60f501) (revert, Reverted, 2022-09-15) - used in the review
- P18: [preserve timestamps when installing headers](https://commitfest.postgresql.org/patch/3363/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P19: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P20: [Range phrase operator in tsquery](https://commitfest.postgresql.org/patch/1624/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P21: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P22: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P23: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P24: [Revert recent changes to 002_pg_upgrade.pl.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=87e22f675fd81ba1d96b0b9a34bbf26d5ec532d5) (revert, Reverted, 2022-08-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 161 s; cost reported by the CLI: $4.37
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 87 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:42:01.930Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
