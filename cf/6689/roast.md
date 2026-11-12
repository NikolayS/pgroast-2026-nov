# Design pre-review: Extensible sync handler registration

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6689](https://commitfest.postgresql.org/patch/6689/); [thread](https://www.postgresql.org/message-id/flat/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No surviving objection is fatal, and the goal follows the custom rmgr pattern. Demand is the main open point: the fsync validation use case does not follow from dispatch by FileTag.handler (demand-1), which leaves Lamstore as the only consumer (demand-2). One design point also stands: handler IDs come from load order and no runtime check catches a mismatch between processes (replication-ondisk-1). Only the author has posted, so this rests on thin material.

Not checked: The code was not read. Whether dispatch checks a handler ID against the table size in non-Assert builds, and how v2 orders registration in EXEC_BACKEND children, could not be confirmed.

## Prior attempts

- [Custom storage managers (SMGR), redux](https://www.postgresql.org/message-id/flat/CAEze2WgMySu2suO_TLvFyGY3URa4mAx22WeoEicnK=PCNWEMrA@mail.gmail.com) (related; Returned with feedback, 2024-10-07). "Quite clearly, the current code is wrong in assuming that the catalog is accessible, and it should probably be stored in a way similar to pg_filenode.map in a file managed outside the buffer pool." ([message](https://www.postgresql.org/message-id/CAEze2WhJaCmXmX%2B%3DkYKU7AY%3DqPAMtx7j-1KFGwAuveRCW%3DstFA%40mail.gmail.com)) Different now: P5 made smgrsw[] a registry selected per tablespace through the catalog. This patch makes only syncsw[] dynamic, with no catalog lookup, so the recorded catalog-in-recovery reason does not apply.
- [Extensible storage manager API - smgr hooks](https://www.postgresql.org/message-id/flat/CAP4vRV6JKXyFfEOf=n+v5RGsZywAQ3CTM8ESWvgq+S87Tmgx_g@mail.gmail.com) (related; Returned with feedback, 2022-03-22). "There has been no activity on this thread for > 6 months. Therefore I'm marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/20220322002038.cno62fjysmw3fc66%40alap3.anarazel.de)) Different now: P6 added an smgr hook for the storage manager table. This patch registers sync handlers only. P6 ended for inactivity, so no design reason carries over.
- [Checkpoint extension hook](https://www.postgresql.org/message-id/flat/CANbhV-E4pTWeF-DsdaGsOrjJNFWPaR+Dstjrnkqvf9JFFgOKKQ@mail.gmail.com) (related; Returned with feedback, 2024-07-23). reason not recorded Different now: P15 proposed a general checkpoint hook for extensions. This patch adds entries to the existing fsync request table. No reason for P15's outcome is recorded.

## Objections a reviewer is likely to raise

### 1. The fsync validation use case is not served by this design

A developer writes the fsync validation extension, registers a handler and gets ID 5. The checkpointer sends it only requests whose FileTag.handler is 5. Requests from md, CLOG and multixact still go to handlers 0 to 4, so the extension sees none of the missing-fsync cases it was built to find.

Message #1 counts the needs that Heikki Linnakangas and Andres Freund mentioned as demand and says a validation extension would "observe every sync request". The proposed dispatch is syncsw[idx], where idx comes from the requester's FileTag, so a registered handler gets only its own requests. The proposal adds no observer or hook on ProcessSyncRequests. Without that use case, the demand rests on one extension.

**Question:** How would a handler registered at ID 5 see sync requests that md.c and slru.c queue under IDs 0 to 4?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com); `demand-1`</sub>

### 2. Demand rests on one unreleased extension from the author

Core adds a public SyncOps API and a documentation chapter for one consumer, Lamstore. Later changes to sync.c, such as a new built-in handler or a change to FileTag, must keep this API stable for extensions that may not exist. If Lamstore changes direction, core still carries the API.

The material names one consumer, the author's Lamstore, and gives no release, user count or production numbers. Only the author has posted in the thread, across three CommitFests (messages #1 to #3). The Percona branch note in message #1 points to the same deployment, so it is not independent demand. The benefit goes to extension authors, but the API surface and enum change apply to everyone.

**Question:** Which other extensions, table AMs or storage projects need checkpoint fsync of non-md files today, and how do they handle it now?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), [#2](https://www.postgresql.org/message-id/177699144837.2925.1505658927335705957%40lamco.io), [#3](https://www.postgresql.org/message-id/20260730214611.2772955.lamco%40lamco.io); `demand-2`</sub>

### 3. Handler IDs from load order are not checked to match across processes

On an EXEC_BACKEND build such as Windows, one extension's _PG_init skips register_sync_handler() in some processes, for example based on process type or a GUC. A backend then queues a FileTag with handler 6, and the checkpointer resolves 6 to another extension's SyncOps or to nothing. Pages are reported synced but never fsynced, which shows up only as data loss after an OS crash.

Message #2 says each EXEC_BACKEND child re-runs shared_preload_libraries and assigns IDs again in its own address space. The design assumes every process reaches the same IDs, but nothing enforces it. The name argument to register_sync_handler() is not used to cross-check IDs, and the commit message lists only Asserts for bounds. The RegisterCustomRmgr parallel in message #1 does not cover this, because rmgr IDs are fixed constants chosen by the extension and are not assigned at runtime.

**Question:** What makes the checkpointer reject, rather than silently misroute, a FileTag whose handler ID maps to a different registered name, or to no name, in a non-Assert build?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), [#2](https://www.postgresql.org/message-id/177699144837.2925.1505658927335705957%40lamco.io); `replication-ondisk-1`</sub>

## Questions for the author

1. How would a handler registered at ID 5 see sync requests that md.c and slru.c queue under IDs 0 to 4?
2. Which extensions, table AMs or storage projects other than Lamstore need checkpoint fsync of non-md files today, and how do they handle it now?
3. In a non-Assert build, what does the checkpointer do with a FileTag whose handler ID is unregistered or maps to a different handler name in that process?
4. Is the name argument of register_sync_handler() checked for uniqueness or used to verify that every process assigned the same ID to the same handler?

## Touch points

- **WAL / replication / standby:** see objection replication-ondisk-1: Handler IDs from load order are not checked to match across processes
- **ABI / extension API:** Adds register_sync_handler(), makes SyncOps public in storage/sync.h, and changes SYNC_HANDLER_NONE from 5 to -1.
- **maintenance burden ("who maintains this in 10 years"):** The checkpoint fsync dispatch table and its FileTag conventions become a documented extension interface.
- **performance on the hot path:** syncsw[] becomes a heap pointer read in ProcessSyncRequests, SyncPostCheckpoint and RememberSyncRequest; the author reports one extra load per call.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the checkpointer fsync queue in sync.c and process startup under EXEC_BACKEND. None has posted in the thread.
- Priority: needs champion; audience niche; review cost medium. The API is small and serves authors of non-md storage extensions, with one named consumer so far. The entry has been through three CommitFests with no review, and the open points need someone who knows sync.c to weigh in.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
