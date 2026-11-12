# Design pre-review: Timing issue in pg_basebackup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7613](https://commitfest.postgresql.org/patch/7613/); [thread](https://www.postgresql.org/message-id/flat/ddfa03dcf26f366ae493e8cc41bd479a%40stjohn.se), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is reproduced on v16 to v18 and a reviewer confirmed it in message #2, so the goal is not in question. One objection survived, maintenance-fit-1: the thread does not compare this client-side reordering with a server-side WAL reservation in BASE_BACKUP. That is a fixable design point and does not block the idea. What remains is back-patch scope and the exit-time slot drop, which need a committer who knows pg_basebackup.

Not checked: Without the code, nobody can judge whether the check in drop_lingering_replication_slot_atexit() is right, or how the handler treats --slot and --create-slot. Only 3 of 6 thread messages were shown.

## Prior attempts

None found before 2026-10-19.

## Objections a reviewer is likely to raise

### 1. Client-side reordering leaves the race for other BASE_BACKUP clients

An operator runs an older pg_basebackup binary, or a third-party tool that speaks BASE_BACKUP and streams WAL separately, against a patched server. A concurrent checkpoint removes WAL after the start position, and the backup still fails with "requested WAL segment ... has already been removed". The operator finds out only when the backup fails.

The race exists because the server reports a start position without reserving WAL past it, yet the fix lives in one client. Every other client of the replication protocol has to copy the same reordering and the same cleanup logic. The proposal does not say whether a server-side guarantee was considered, such as BASE_BACKUP holding WAL from its start position until the backup ends. Nothing in the thread (messages #1 to #6) compares the two approaches.

**Question:** Why is a client-side reordering better than having BASE_BACKUP itself keep WAL from the start position, which would also protect older clients and other tools?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/ddfa03dcf26f366ae493e8cc41bd479a%40stjohn.se), [#5](https://www.postgresql.org/message-id/b324489a123ae3b43e2eca9a5833157e%40stjohn.se), [#6](https://www.postgresql.org/message-id/80143a07e6732576c9f815a3058fa31d%40stjohn.se); `maintenance-fit-1`</sub>

## Questions for the author

1. Why is reordering in pg_basebackup better than having BASE_BACKUP keep WAL from its start position, which would also protect older clients and other tools?
2. Which branches is this fix meant for: master only, or also v16 through v18?
3. How does the new order behave with --slot, with --create-slot, and with the default temporary slot, and which of these does the atexit handler drop?
4. What does the atexit handler do when the connection is already lost, or when the slot drop itself fails?
5. Does the new order work against every older server version that pg_basebackup still supports?

## Touch points

- **WAL / replication / standby:** The replication slot now exists before BASE_BACKUP starts, so it holds WAL for longer, mainly with --checkpoint=spread.
- **backpatchability:** The bug is reported on v16, v17 and v18, but the proposal names no target branches.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds an atexit handler in pg_basebackup.c that drops the slot, plus an injection-point TAP test.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows pg_basebackup and replication slot handling; none has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost low. The race affects pg_basebackup --wal-method=stream when a checkpoint runs at the wrong moment, and it is reproduced on three releases. The patch is about 13 KiB in one client program plus tests, with no new user-visible option.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
