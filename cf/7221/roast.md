# Design pre-review: pg_rewind: file sync bypass and findLastCheckpoint boundary crash

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7221](https://commitfest.postgresql.org/patch/7221/); [thread](https://www.postgresql.org/message-id/flat/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). Level 1b is where a committer would stop. already-exists-1 says a cleanly stopped old primary started as a standby follows the new timeline today, and the thread shows only a postgresql.auto.conf difference. standards-compat-1 says the planned switchover changes from no writes to a file copy, and the thread does not weigh that. Only the author has posted, so the verdict rests on what the thread does not yet show.

Not checked: Without the code, I could not check how the fallback detects end of written WAL, where the forward scan stops, or which files the sync copies in this case. The linked thread from message #2 is not in the material.

## Prior attempts

- [Fix pg_rewind false positives caused by shutdown-only WAL](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com) (related; Withdrawn, 2026-05-01). "But I do not think we can do that, because (1) those WAL records might do important things like update the control file and (2) the server will not be OK with ignoring those WAL records even if pg_rewind decides that they are not important." ([message](https://www.postgresql.org/message-id/CA%2BTgmoYYvYShoRKK6%3D8PX-BYBarFXjrGB%2BcBBRjX-SkMfd0Nrg%40mail.gmail.com)) Different now: P8 made pg_rewind skip shutdown-only WAL so that fewer cases need a rewind. This patch does the opposite: it runs the file sync and updates pg_control in a case that used to exit early, which agrees with P8's objection.

## Objections a reviewer is likely to raise

### 1. Clean switchover already works by starting the old primary as standby

After a clean switchover, an operator adds standby.signal and primary_conninfo to the old primary and starts it, and it follows the new timeline. With the patch, a pg_rewind run in the same state copies files and rewrites pg_control through a new forward-scan path. A regression there breaks a switchover that needed no rewind at all.

In this case the old primary's WAL ends at the divergence point, so it can follow the new timeline. Recovery fetches the timeline history file from the new primary or the archive and switches timelines itself. Message #1 lists the missing history file, pg_control state and postgresql.auto.conf, but the first two are already handled by recovery. The third is a configuration difference, and standbys often keep their own settings on purpose. So the gap is convenience in tooling that always runs pg_rewind, and the material does not show anything that cannot be done today.

**Question:** What concrete failure does a user hit when the old primary is started as a standby without pg_rewind after a clean switchover, which recovery cannot already handle?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com); `already-exists-1`</sub>

### 2. Planned switchover now rewrites the old primary's files and pg_control

An HA tool stops the primary cleanly, promotes a standby, then runs pg_rewind as a routine step. Today pg_rewind reports that no rewind is required and leaves the old primary alone. After the patch it overwrites postgresql.auto.conf and other config files with the new primary's copies and rewrites pg_control, and operators find out when the old primary starts with settings they did not choose.

A clean stop followed by promotion is the normal planned switchover, so this is the common path. Message #1 treats copying postgresql.auto.conf as the desired result. But in this case some setups may rely on pg_rewind leaving the old primary untouched. The patch touches no documentation files, so neither the pg_rewind docs nor any release note would describe the change.

**Question:** Which tools or scripts run pg_rewind after a clean switchover, and does each of them expect the old primary's config files to be overwritten?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6qpqH4E22-PAemSby8Yv_2Z_8PUy8XrUmpW9Ot_efKdug%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What concrete failure does a user hit when a cleanly stopped old primary starts as a standby without pg_rewind, which recovery cannot already handle?
2. Was the early exit for target_wal_endrec == divergerec intended, and do any tools rely on pg_rewind leaving the target untouched in that case?
3. Which files does the sync copy in this case, and why is copying the source's postgresql.auto.conf over the target's the wanted result?
4. Why not take the last common checkpoint from the target's pg_control when target_wal_endrec equals divergerec, instead of adding a forward scan to findLastCheckpoint()?
5. How does the fallback tell end of written WAL from a missing or damaged segment at forkptr, and what happens if the forward scan finds no checkpoint?

## Touch points

- **on-disk format / pg_upgrade:** pg_rewind now rewrites the target's pg_control and copies files in a case where it wrote nothing before.
- **WAL / replication / standby:** The patch changes how findLastCheckpoint() reads target WAL and how an old primary is prepared to follow a new timeline.
- **backpatchability:** The entry is tagged Bugfix but changes visible pg_rewind behaviour; it sets target version 20 and does not mention back branches.
- **maintenance burden ("who maintains this in 10 years"):** findLastCheckpoint() gains a second, error-triggered forward-scan mode that later xlogreader changes must keep working.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Starting the cleanly stopped old primary as a standby may already cover this case without pg_rewind.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pg_rewind, timeline switching and the xlogreader code. Only the author has posted in the thread.
- Priority: fix design first; audience unclear; review cost medium. The patch is small and touches only pg_rewind, but it changes the result of the common planned-switchover case and reads WAL in a new way. The thread must first show what fails today and whether the checkpoint can come from pg_control.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
