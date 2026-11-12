# Design pre-review: Extensible sync handler registration

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6689](https://commitfest.postgresql.org/patch/6689/) (Needs review); authors: Greg Lamberson (glamberson)
- Thread: [Extensible sync handler registration (register_sync_handler)](https://www.postgresql.org/message-id/flat/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), started 2026-04-10
- Messages read: 3 of 3
- Patch prose read: v2-0001-Make-sync.c-syncsw-extensible-via-register_sync_h.patch, v2-0002-Add-test-module-for-sync-handler-registration.patch, v1-0001-Make-sync.c-syncsw-extensible-via-register_sync_h.patch, v1-0002-Add-test-module-for-sync-handler-registration.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No surviving objection is fatal, and the goal follows the custom rmgr pattern. Demand is the main open point: the fsync validation use case does not follow from dispatch by FileTag.handler (demand-1), which leaves Lamstore as the only consumer (demand-2). One design point also stands: handler IDs come from load order and no runtime check catches a mismatch between processes (replication-ondisk-1). Only the author has posted, so this rests on thin material.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether dispatch checks a handler ID against the table size in non-Assert builds, and how v2 orders registration in EXEC_BACKEND children, could not be confirmed.

## The proposal, restated

**Problem.** Extensions cannot add entries to the checkpoint fsync dispatch table, because syncsw[] in sync.c is a fixed static array. The author's extent-based storage extension, Lamstore, keeps data at byte offsets inside a volume file, so it cannot reuse SYNC_HANDLER_MD, which fsyncs md segment paths. The author also cites fsync validation tools that Heikki Linnakangas and Andres Freund asked for in the CF 5616 thread.

**Proposed design.** syncsw[] becomes a heap-allocated table that grows by repalloc. InitSyncHandlers() registers the five built-in handlers at their existing IDs 0 to 4 before shared_preload_libraries load; under EXEC_BACKEND, register_sync_handler() also calls it. Extensions call register_sync_handler() from _PG_init, get an int16 ID from SYNC_HANDLER_FIRST_DYNAMIC on, and put it in FileTag.handler.

**User-visible surface.**
- New C function register_sync_handler(const SyncOps *ops, const char *name) returning int16; calls after preload raise FATAL.
- SyncOps struct moves to the public header storage/sync.h.
- SyncRequestHandler enum: new SYNC_HANDLER_FIRST_DYNAMIC = 5 and SYNC_HANDLER_MAX; SYNC_HANDLER_NONE changes from 5 to -1.
- New documentation chapter custom-sync-handler.sgml, modeled on custom-rmgr.sgml.
- New test module src/test/modules/test_sync_handler with a TAP test.

**What the author claims.**
- The change is strictly additive, with no changes to WAL format, FileTag layout, shared memory or catalogs, and no behaviour change without a registered extension.
- Per-dispatch assembly is byte identical to stock on GCC 14.2 -O2; the only cost is one extra load per ProcessSyncRequests call.
- Commit 3eb77eba5a5 (PG12) planned for non-md sync handler users, so this patch adds the missing registration step.
- The patch is independent of CF 5616 (extensible smgr) and composes with it.
- v2 passes check-world on fork and EXEC_BACKEND builds, and CFbot is green on all platforms.

**Thread state.** Only the author has posted: v1, a v2 that fixes EXEC_BACKEND registration order, and a status ping saying no review yet. A reviewer signed up in the CommitFest on 2026-11-09 but has not posted.

## 2. Prior attempts

### Custom storage managers (SMGR), redux (related)

- **Link:** [https://commitfest.postgresql.org/patch/4428/](https://commitfest.postgresql.org/patch/4428/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAEze2WgMySu2suO_TLvFyGY3URa4mAx22WeoEicnK=PCNWEMrA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAEze2WgMySu2suO_TLvFyGY3URa4mAx22WeoEicnK=PCNWEMrA@mail.gmail.com)
- **Date:** decided 2024-10-07; started 2023-06-30
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Kirill Reshke (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch stayed incomplete. Tablespace redo was broken, and the code looked up a tablespace's storage manager in the catalog, which is not readable during crash recovery. The author agreed and planned a file outside the buffer pool, but no revised version was posted after the co-CFM reported cfbot failures in March 2024. In the last message a reviewer asked to settle hook-based versus registration-based design first.
- **Reason (quote):** "Quite clearly, the current code is wrong in assuming that the catalog is accessible, and it should probably be stored in a way similar to pg_filenode.map in a file managed outside the buffer pool." ([message](https://www.postgresql.org/message-id/CAEze2WhJaCmXmX%2B%3DkYKU7AY%3DqPAMtx7j-1KFGwAuveRCW%3DstFA%40mail.gmail.com))
- **What is different this time:** P5 made smgrsw[] a registry selected per tablespace through the catalog. This patch makes only syncsw[] dynamic, with no catalog lookup, so the recorded catalog-in-recovery reason does not apply.

### Extensible storage manager API - smgr hooks (related)

- **Link:** [https://commitfest.postgresql.org/patch/3216/](https://commitfest.postgresql.org/patch/3216/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAP4vRV6JKXyFfEOf=n+v5RGsZywAQ3CTM8ESWvgq+S87Tmgx_g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAP4vRV6JKXyFfEOf=n+v5RGsZywAQ3CTM8ESWvgq+S87Tmgx_g@mail.gmail.com)
- **Date:** decided 2022-03-22; started 2021-06-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Andres Freund (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The thread had no activity for more than six months, so Andres Freund marked it returned with feedback and said it could not be merged into 15. Yura Sokolov's only review asked how an implementation is chosen per relation and suggested tying it to the tablespace with a less invasive change. The visible messages show no reply from the author.
- **Reason (quote):** "There has been no activity on this thread for > 6 months. Therefore I'm marking it as returned with feedback." ([message](https://www.postgresql.org/message-id/20220322002038.cno62fjysmw3fc66%40alap3.anarazel.de))
- **What is different this time:** P6 added an smgr hook for the storage manager table. This patch registers sync handlers only. P6 ended for inactivity, so no design reason carries over.

### Checkpoint extension hook (related)

- **Link:** [https://commitfest.postgresql.org/patch/4868/](https://commitfest.postgresql.org/patch/4868/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANbhV-E4pTWeF-DsdaGsOrjJNFWPaR+Dstjrnkqvf9JFFgOKKQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANbhV-E4pTWeF-DsdaGsOrjJNFWPaR+Dstjrnkqvf9JFFgOKKQ@mail.gmail.com)
- **Date:** decided 2024-07-23; started 2022-05-11
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P15 proposed a general checkpoint hook for extensions. This patch adds entries to the existing fsync request table. No reason for P15's outcome is recorded.


## 3. Objections a reviewer is likely to raise

### 1. The fsync validation use case is not served by this design

fixable · level 1b, rubric item 2 · demand, ABI / extension API · `demand-1`

A developer writes the fsync validation extension, registers a handler and gets ID 5. The checkpointer sends it only requests whose FileTag.handler is 5. Requests from md, CLOG and multixact still go to handlers 0 to 4, so the extension sees none of the missing-fsync cases it was built to find.

Message #1 counts the needs that Heikki Linnakangas and Andres Freund mentioned as demand and says a validation extension would "observe every sync request". The proposed dispatch is syncsw[idx], where idx comes from the requester's FileTag, so a registered handler gets only its own requests. The proposal adds no observer or hook on ProcessSyncRequests. Without that use case, the demand rests on one extension.

Thread: [#1 (Greg Lamberson, 2026-04-10)](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com)

**Question:** How would a handler registered at ID 5 see sync requests that md.c and slru.c queue under IDs 0 to 4?

Counter-check: held. The proposal dispatches by FileTag.handler and adds no observer on ProcessSyncRequests, so a registered handler receives only its own requests. The claim in message #1 that a validation extension observes every sync request does not follow from the design.

### 2. Demand rests on one unreleased extension from the author

fixable · level 1b, rubric item 3 · demand, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `demand-2`

Core adds a public SyncOps API and a documentation chapter for one consumer, Lamstore. Later changes to sync.c, such as a new built-in handler or a change to FileTag, must keep this API stable for extensions that may not exist. If Lamstore changes direction, core still carries the API.

The material names one consumer, the author's Lamstore, and gives no release, user count or production numbers. Only the author has posted in the thread, across three CommitFests (messages #1 to #3). The Percona branch note in message #1 points to the same deployment, so it is not independent demand. The benefit goes to extension authors, but the API surface and enum change apply to everyone.

Thread: [#1 (Greg Lamberson, 2026-04-10)](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), [#2 (Greg Lamberson, 2026-04-24)](https://www.postgresql.org/message-id/177699144837.2925.1505658927335705957%40lamco.io), [#3 (Greg Lamberson, 2026-07-30)](https://www.postgresql.org/message-id/20260730214611.2772955.lamco%40lamco.io)

**Question:** Which other extensions, table AMs or storage projects need checkpoint fsync of non-md files today, and how do they handle it now?

Counter-check: held. The material names Lamstore as the only working consumer, and only the author has posted across three CommitFests. The 2019 commit message mentions other proposed users, but the thread shows none today.

### 3. Handler IDs from load order are not checked to match across processes

fixable · level 2, rubric item 7 · ABI / extension API, WAL / replication / standby · `replication-ondisk-1`

On an EXEC_BACKEND build such as Windows, one extension's _PG_init skips register_sync_handler() in some processes, for example based on process type or a GUC. A backend then queues a FileTag with handler 6, and the checkpointer resolves 6 to another extension's SyncOps or to nothing. Pages are reported synced but never fsynced, which shows up only as data loss after an OS crash.

Message #2 says each EXEC_BACKEND child re-runs shared_preload_libraries and assigns IDs again in its own address space. The design assumes every process reaches the same IDs, but nothing enforces it. The name argument to register_sync_handler() is not used to cross-check IDs, and the commit message lists only Asserts for bounds. The RegisterCustomRmgr parallel in message #1 does not cover this, because rmgr IDs are fixed constants chosen by the extension and are not assigned at runtime.

Thread: [#1 (Greg Lamberson, 2026-04-10)](https://www.postgresql.org/message-id/IA1PR07MB983072521EE7FDEE98902534A9592%40IA1PR07MB9830.namprd07.prod.outlook.com), [#2 (Greg Lamberson, 2026-04-24)](https://www.postgresql.org/message-id/177699144837.2925.1505658927335705957%40lamco.io)

**Question:** What makes the checkpointer reject, rather than silently misroute, a FileTag whose handler ID maps to a different registered name, or to no name, in a non-Assert build?

Counter-check: held. Message #2 confirms each EXEC_BACKEND child assigns IDs again, and the commit message lists only Asserts as bounds checks. The trigger is an extension that registers conditionally, but the result is a silently skipped fsync, so a runtime check is a fair request.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | see objection replication-ondisk-1: Handler IDs from load order are not checked to match across processes |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Adds register_sync_handler(), makes SyncOps public in storage/sync.h, and changes SYNC_HANDLER_NONE from 5 to -1. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The checkpoint fsync dispatch table and its FileTag conventions become a documented extension interface. |
| performance on the hot path | **yes** | syncsw[] becomes a heap pointer read in ProcessSyncRequests, SyncPostCheckpoint and RememberSyncRequest; the author reports one extra load per call. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How would a handler registered at ID 5 see sync requests that md.c and slru.c queue under IDs 0 to 4?
2. Which extensions, table AMs or storage projects other than Lamstore need checkpoint fsync of non-md files today, and how do they handle it now?
3. In a non-Assert build, what does the checkpointer do with a FileTag whose handler ID is unregistered or maps to a different handler name in that process?
4. Is the name argument of register_sync_handler() checked for uniqueness or used to verify that every process assigned the same ID to the same handler?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the checkpointer fsync queue in sync.c and process startup under EXEC_BACKEND. None has posted in the thread.
- **Liability:** The owner signs for checkpoint fsync correctness with third-party handlers in the table, and for keeping SyncOps and handler ID assignment working as sync.c changes. A misrouted request shows up only as data loss after an OS crash.
- **Author follow-up:** Good signs: the author posted v2 two weeks after v1 with an EXEC_BACKEND fix, sent a status update in message #3, and offered to rework the design.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The API is small and serves authors of non-md storage extensions, with one named consumer so far. The entry has been through three CommitFests with no review, and the open points need someone who knows sync.c to weigh in.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Sync handler API fixed before the extensible smgr design settles. Refuted (no failure scenario): The scenario does not say how a hook-based or per-tablespace smgr would change sync dispatch, which core already indexes by FileTag.handler. [P6] closed for inactivity and [P5] for catalog and redo problems, neither of which touches sync.c.
- `maintenance-fit-1` Fsync validation use case does not follow from the design. Refuted (not specific): Duplicate of demand-1, which states the same dispatch fact more narrowly.
- `maintenance-fit-2` Fixes a sync-side registration API before smgr extensibility is designed. Refuted (no failure scenario): Same speculative scenario as prior-attempts-1. Commit 3eb77eba5a5, quoted in message #1, kept the sync table separate from smgr on purpose, and message #1 says Lamstore's storage manager already works on stock Postgres.
- `maintenance-fit-3` FileTag and handler lifecycle become a public contract with unclear rules. Refuted (not specific): FileTag and RegisterSyncRequest are already in the public storage/sync.h, and any extension API can change across major versions. The unknown handler ID point is covered by replication-ondisk-1.

### Review angles

- prior-attempts: 1 objection(s). No precedent is the same idea. The thread contains no reviewer reply about earlier attempts. Message #1 points to the earlier smgr_register review, and the message reports that review only by analogy.
- standards-compat: 0 objection(s). No standards or compatibility objection holds. The patch adds no SQL, GUC, catalog object, dump state or psql surface, and the change of SYNC_HANDLER_NONE to -1 is flagged in message #2 and lands only in a major release. The open points are safety and design questions: unknown handler IDs, whether one handler can observe another's requests, and FileTag conventions. Other angles cover those.
- replication-ondisk: 1 objection(s). The proposal adds no WAL records, catalogs or on-disk format, and sync requests are in memory only, so pg_upgrade and replication are not affected. The SYNC_HANDLER_NONE change targets a new major version, so the ABI break is acceptable there.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. register_sync_handler() is a C API callable only from _PG_init during shared_preload_libraries, which only a superuser or the server owner can set, and such code already runs with full server rights. No SQL surface, role, or view is added. Dispatch of sync requests with unknown handler IDs is a safety question for the design angle.
- maintenance-fit: 3 objection(s). Hot-path cost is not a concern: the extra load happens once per checkpointer call, and only on the checkpointer. No committer has engaged in three CommitFests, which affects committability, not the verdict.
- already-exists: 0 objection(s). No core feature or extension does this today. syncsw[] is fixed, SYNC_HANDLER_MD only fsyncs md segment paths, and no checkpoint hook exists ([P15] was returned), so the only fallback is a synchronous fsync on every write, which cannot replace checkpoint-time sync at an acceptable cost. The patch extends the existing sync.c mechanism, following the custom rmgr pattern, rather than adding a second one.
- demand: 2 objection(s). Two demand objections; the API is small, so the main gap is evidence of users beyond Lamstore.

### Precedents retrieved

- P1: [Custom signals and handlers for extension](https://commitfest.postgresql.org/patch/1430/) (cf, Returned with feedback, 2020-11-26) - judged not relevant; reason summarised from its thread
- P2: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P3: [Fsync queue forgets requests on failure](https://commitfest.postgresql.org/patch/1613/) (cf, Rejected, 2018-07-13) - judged not relevant; reason summarised from its thread
- P4: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P6: [Extensible storage manager API - smgr hooks](https://commitfest.postgresql.org/patch/3216/) (cf, Returned with feedback, 2022-03-22) - used in the review; reason summarised from its thread
- P7: [Revert removal of signed, volatile, and signal handler arg type tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ef6164de1d60f65039a73f5975d50b4fa10b8a5b) (revert, Reverted, 2000-08-29) - judged not relevant
- P8: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P10: [Revert initdb --sync-only patch that had incorrect commit messages.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=25d1ed04a2c4d176f6428b9eed57344e50f44d77) (revert, Reverted, 2012-12-04) - judged not relevant
- P11: [Allow synchronous_standby_names to be disabled after communication failure with all synchronous standby servers exceeds some timeout](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Configuration_files) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P12: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P13: [Inconsistency in startup process's MyBackendId and procsignal array registration with ProcSignalInit()](https://commitfest.postgresql.org/patch/3357/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P14: [Add minimal C example and SQL registration example for custom table access methods](https://commitfest.postgresql.org/patch/4973/) (cf, Withdrawn, 2024-07-25) - judged not relevant
- P15: [Checkpoint extension hook](https://commitfest.postgresql.org/patch/4868/) (cf, Returned with feedback, 2024-07-23) - used in the review
- P16: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant
- P17: [new diagnostic items for the dynamic sql](https://commitfest.postgresql.org/patch/3537/) (cf, Returned with feedback, 2022-12-14) - judged not relevant
- P18: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P19: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/1361/) (cf, Returned with feedback, 2017-11-30) - judged not relevant
- P20: [Dynamic result sets from procedures](https://commitfest.postgresql.org/patch/2911/) (cf, Withdrawn, 2023-10-30) - judged not relevant
- P21: [Per-table storage parameters for TableAM/IndexAM extensions](https://commitfest.postgresql.org/patch/3495/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P22: [asynchronous execution support for Custom Scan](https://commitfest.postgresql.org/patch/3813/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P23: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P24: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 175 s; cost reported by the CLI: $2.57
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 87 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:00:02.004Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
