# Design pre-review: Warn when creating or enabling a subscription with max_logical_replication_workers = 0

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6642](https://commitfest.postgresql.org/patch/6642/); [thread](https://www.postgresql.org/message-id/flat/20260204140731.0e4b511906ac748abad1f3d9%40sraoss.co.jp), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated, the cost falls only on misconfigured servers, and reviewers found the warning useful. The one surviving objection, maintenance-fit-1, is a fixable design point about where the check lives. Message #13 already shows a gap between CREATE and ALTER ... SET (retain_dead_tuples), so a committer would likely ask for one shared test of whether the launcher runs.

Not checked: The code was not read, so the exact warning conditions, the behaviour during a pg_dump restore, and how the check relates to ApplyLauncherRegister() could not be confirmed. Only 3 of 13 messages were shown in full.

## Prior attempts

None found before 2026-02-04.

## Objections a reviewer is likely to raise

### 1. Warning condition copied into command sites, so it drifts from launcher.c

A later patch adds a subscription option that needs the launcher, or changes when the launcher registers. The patch author updates launcher.c and calls ApplyLauncherWakeupAtCommit(), but does not add the new check in subscriptioncmds.c. Users on a server with the launcher absent then get silence again, and no test fails.

The check tests max_logical_replication_workers directly in subscriptioncmds.c, while ApplyLauncherRegister() in launcher.c decides on its own conditions (message #5). The gaps already exist. Message #13 shows that ALTER ... SET (retain_dead_tuples) does not warn and that max_worker_processes = 0 is silent. Peter Smith raised the other callers in message #9, and the author kept the per-site approach in message #10. No test exercises the path, so drift will not show up.

**Question:** Can the decision whether the launcher will run come from one function in launcher.c, so that new callers and new registration conditions cannot miss the warning?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CAHut%2BPto3FdeAaQJgcXZHk8qOnGOTpWW7K98mxzVckgQjWWXHw%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/CAHut%2BPtmxmkP4acNT0b8b49vySXSEAARvDFYuqbyBWBho4iH%2Bg%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/20260413160405.a22d5a0111c612176ac9eb3f%40sraoss.co.jp), [#13](https://www.postgresql.org/message-id/CADMs5OO%2B1h1LhASBY0zMoTyUeWfpp-_wCD6azMtRT_%3D0ym4A1A%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Can one function in launcher.c report whether the launcher will run, so that every command site uses the same test?
2. Should ALTER SUBSCRIPTION ... SET (retain_dead_tuples) warn, given that CREATE SUBSCRIPTION warns for the same option?
3. Should the warning also cover max_worker_processes = 0, where the launcher is also absent and the commands succeed silently?
4. Will the next version add a regression or TAP test that fails when a command site misses the warning?
5. Does a pg_dump restore of a subscription with retain_dead_tuples = true emit the warning on a server with the setting at 0, and is that intended?

## Touch points

- **WAL / replication / standby:** The warning reports that the logical replication launcher will not start on the subscriber; no WAL or protocol behaviour changes.
- **pg_dump / restore round-trip:** A restore that runs CREATE SUBSCRIPTION with retain_dead_tuples = true on a server with the setting at 0 may emit the new warning.
- **maintenance burden ("who maintains this in 10 years"):** The check in subscriptioncmds.c repeats a condition that launcher.c decides, and no test covers it.

## Who would commit it, and when

- Owner: silent. Someone who knows the logical replication launcher and subscription commands would have to own it; the visible thread and CommitFest entry show no committer engaged.
- Priority: proceed; audience niche; review cost low. The patch touches one file and only helps users who set max_logical_replication_workers to 0, which message #6 calls rare. It needs a rebase, a test and an answer on where the check lives before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
