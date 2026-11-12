# Design pre-review: Two bugs around ALTER TYPE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6903](https://commitfest.postgresql.org/patch/6903/); [thread](https://www.postgresql.org/message-id/flat/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Both bugs are reported by users and reproduced by a reviewer, and 0002 uses the existing relationOids invalidation channel. The surviving objections are fixable design points about the 0004 lock: its duration and back-branch status (standards-compat-1), its PL/pgSQL-only scope (standards-compat-2) and its effect on hot standby replay (replication-ondisk-1). None questions the goal. A committer who knows locking and the plan cache has to judge the new blocking behavior.

Not checked: Without the code, it is not possible to check that 0002 covers all paths that build function RTE dependencies, or exactly when 0004 takes the lock relative to the cached descriptor.

## Prior attempts

- [Support prepared statement invalidation when result or argument types change](https://www.postgresql.org/message-id/flat/CAGECzQQ3nPt5joW50FygoM_7YU9QjKO06say4c-sCceqh8vnsg@mail.gmail.com) (similar idea; Withdrawn, 2024-07-25). "What if the client has previously done a Describe Statement on the prepared statement? We have no mechanism for notifying it that that information is now falsified." ([message](https://www.postgresql.org/message-id/1238027.1721835596%40sss.pgh.pa.us)) Different now: P2 removed the "cached plan must not change result type" check so clients would get a new RowDescription. This proposal only adds the composite typrelid to plan dependencies and does not change what is reported to clients, so the reason P2 was withdrawn does not apply.

## Objections a reviewer is likely to raise

### 1. New lock changes ALTER TYPE blocking for existing users, mainly if backpatched

An application keeps a transaction open after calling a SETOF PL/pgSQL function. A deploy script runs ALTER TYPE on the standalone composite type and waits on its AccessExclusiveLock. Every new call to any SETOF function returning that type then queues behind it, and the script that used to finish at once now stalls the application.

Patch 0004 takes AccessShareLock at function entry and holds it to transaction end. Message #1 justifies it as keeping the row shape stable within one statement, which is a shorter span than the lock covers. Today, ALTER TYPE on a standalone composite does not wait for function callers, so this is a new blocking contract. The material does not say whether the fix targets back branches, where such a change in locking behavior would surprise users after a minor upgrade.

**Question:** Is the lock meant for back branches, and why must it last to transaction end instead of the end of the calling statement?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), [#5](https://www.postgresql.org/message-id/69E6F43A-76A1-4A68-B03E-B6CC769944FC%40yandex-team.ru); `standards-compat-1`</sub>

### 2. Race fix covers only PL/pgSQL, other function languages keep the race

A user ports the reported function to a SQL-language or PL/Python SETOF function returning the same composite. A concurrent ALTER TYPE plus CREATE OR REPLACE FUNCTION commits mid-call. The user hits the same row-shape mismatch error that the release notes say is fixed.

Message #1 places the root cause in plpgsql_estate_setup(), and patch 0004 touches only pl_exec.c. The condition it guards against is a tuple descriptor captured at call start and then replanned against a changed type. That condition is not specific to PL/pgSQL. A committer would likely ask for a fix in the common function-call or SRF path, or for a stated reason why other languages are safe.

**Question:** Do SQL-language functions and other PLs returning SETOF a named composite have the same race, and if so, where would a common fix go?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru); `standards-compat-2`</sub>

### 3. New lock on typrelid adds hot standby replay conflicts

A reporting standby runs a long transaction that calls a SETOF PL/pgSQL function returning planinv_ct. The primary runs ALTER TYPE planinv_ct ADD ATTRIBUTE. Replay of the logged AccessExclusiveLock on typrelid now waits behind the standby's new AccessShareLock, so replay stalls up to max_standby_streaming_delay and then cancels the query.

0004 takes AccessShareLock on typrelid at function entry and holds it to transaction end (message #1). Standbys replay AccessExclusiveLock records, so this creates a conflict that did not exist before, on standbys that never run the DDL. This copies how table scans behave, which lowers the concern, but the thread covers only the primary. If the fix goes to back branches, standby operators would see new replay lag or new query cancels after a minor update.

**Question:** How does the new lock behave on a hot standby during replay of ALTER TYPE, and is that change acceptable in a minor release if this is backpatched?

<sub>fixable, level 2, WAL / replication / standby; messages [#1](https://www.postgresql.org/message-id/39977E94-40DD-4176-A191-CE3D120968B2%40yandex-team.ru), [#4](https://www.postgresql.org/message-id/95599FF6-EDE9-42B2-8940-7856F4A3A199%40yandex-team.ru); `replication-ondisk-1`</sub>

## Questions for the author

1. Are 0002 and 0004 meant for back branches, or only for master?
2. Why must the 0004 lock last to transaction end instead of the end of the calling statement?
3. Do SQL-language functions and other PLs returning SETOF a named composite have the same race, and if so, where would a common fix go?
4. How does the new lock behave on a hot standby when replay reaches ALTER TYPE on the same composite type?
5. What do the extra plan dependency and the per-call lock cost for a workload that calls such functions often?

## Touch points

- **WAL / replication / standby:** A standby transaction holding the new AccessShareLock on typrelid conflicts with replay of ALTER TYPE's AccessExclusiveLock.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: New lock changes ALTER TYPE blocking for existing users, mainly if backpatched
- **planner behaviour change:** 0002 changes setrefs.c so plans scanning composite-returning functions gain a relation dependency and are replanned after ALTER TYPE.
- **backpatchability:** The entry is tagged Bugfix, and 0004 adds a lock that makes ALTER TYPE wait where it did not before.
- **maintenance burden ("who maintains this in 10 years"):** The lock lives only in pl_exec.c, and alter_type.sgml documents a new invalidation rule the project must keep.
- **performance on the hot path:** 0004 takes a lock on every call of a SETOF PL/pgSQL function returning a named composite; no measurement is shown.

## Who would commit it, and when

- Owner: silent. Someone who knows plan cache invalidation and heavyweight lock behavior in PL/pgSQL; Heikki Linnakangas is a listed reviewer but has not posted.
- Priority: needs champion; audience niche; review cost medium. The patches are small and fix reported bugs, but the entry has sat in 3 CommitFests with no committer posting. The 0004 lock needs judgment on blocking behavior; 0002 could go ahead separately.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 2.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
