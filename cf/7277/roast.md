# Design pre-review: Report userid to PgBackendStatus for standalone backends

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7277](https://commitfest.postgresql.org/patch/7277/); [thread](https://www.postgresql.org/message-id/flat/tencent_18F1100EB0ECCA42510B516A835214640409%40qq.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Only demand-1 survived, and it fails level 1b on demand. Message #2 asked who benefits when no other process can read the status of a postgres --single backend, and message #3 answered that the empty user name looks strange but is harmless. The change is small and costs users of normal servers nothing, so this is a "not now, show demand" outcome and no rejection of the idea.

Not checked: Without the code, nobody could check whether bootstrap mode (initdb) takes the same path, or at which point in backend status setup the user ID is recorded.

## Prior attempts

None found before 2026-09-08.

## Objections a reviewer is likely to raise

### 1. No user is shown to need the user name in single-user mode

A DBA runs postgres --single to recover from wraparound and queries pg_stat_activity. Only that one session can see its own row, and the DBA already knows it runs as the bootstrap superuser. The patch gives this user nothing new, and a committer still takes on a change to backend status startup.

Message #2 asked why this matters when no other process can read the status of a postgres --single backend. Message #3 does not name a use. It says the empty user name "looks a bit strange though harmless" and that the author noticed it while querying in single-user mode. The thread contains no field report, no request from another user, and no tool that reads usename in single-user mode. The patch is small, but small cost does not replace a stated user problem.

**Question:** Which tool or recovery procedure reads usesysid or usename from pg_stat_activity in single-user mode, and what goes wrong for it today?

<sub>fixable, level 1b, demand; messages [#2](https://www.postgresql.org/message-id/aqCaBrpR7jsa8yxo%40paquier.xyz), [#3](https://www.postgresql.org/message-id/tencent_787867078376F4A11C8558B57829CF327308%40qq.com); `demand-1`</sub>

## Questions for the author

1. Which tool or recovery procedure reads usesysid or usename from pg_stat_activity in single-user mode, and what goes wrong for it today?
2. Does bootstrap mode (initdb) take the same code path, and what does the patch record there?
3. Are other pg_stat_activity columns also empty or wrong in single-user mode, and why does the patch fill only the user ID?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch adds a case for standalone backends to backend status setup in backend_status.c.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Only the single-user session can read its own pg_stat_activity row, and the thread names no user who needs usename there.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier replied in message #2 and questioned the need; the owner would be someone who knows the backend status and activity reporting code.
- Not now, show demand first: A named tool, monitoring query or recovery procedure that reads usesysid or usename from pg_stat_activity in single-user mode and misbehaves on the empty value, or a report from another user who hit this.
- Priority: park; audience niche; review cost low. The patch changes one file and is 1.1 KiB, so cost is low, but it affects only postgres --single sessions and message #3 calls the current state harmless. It is worth reviewer time once a use is shown.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
