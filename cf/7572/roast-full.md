# Design pre-review: Standby stuck after walreceiver restart just before a WAL segment boundary

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7572](https://commitfest.postgresql.org/patch/7572/) (Needs review); authors: David Shorten (shortenda)
- Thread: [Standby stuck after walreceiver restart just before a WAL segment boundary](https://www.postgresql.org/message-id/flat/CADstiJuSX%3D0KZX%2B99q%3DLH4TtV0my%2Bnr3m93asc1bzn_vOiHqug%40mail.gmail.com), started 2026-10-31
- Messages read: 5 of 5
- Patch prose read: v1-0001-Fix-walreceiver-restart-skipping-the-end-of-a-WAL.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The proposal states a reproducible standby hang, its cause, and a fix that changes behavior only when RecPtr is exactly on a segment boundary. It adds no user-visible surface, format or API. Confidence is medium because the code was not read and no committer has spoken in the visible thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that RecPtr - 1 always falls in the segment of the page being read is unchecked, as is the stability of the test. Messages #4 and #5 were not shown in full.

## The proposal, restated

**Problem.** A streaming standby stops making progress if its walreceiver restarts while the last received record ends 24 bytes or fewer before a segment end. The new walreceiver starts streaming at the next segment, so the tail of the current segment is never fetched and recovery loops on "invalid record length". Users of any supported branch who change primary_conninfo or lose the connection can hit it.

**Proposed design.** WaitForWALToBecomeAvailable() passes RecPtr, the end of the needed WAL, to RequestXLogStreaming(), which rounds it down to a segment start. The patch starts streaming at the segment holding RecPtr - 1, the segment of the page being read. Behavior changes only when RecPtr is exactly on a segment boundary.

**What the author claims.**
- The bug reproduces on master, and the code is the same on all supported branches.
- The fix matches what commit c06380e976 intended when it switched to RecPtr.
- Since b614de4876 the standby hangs silently; before that, START_REPLICATION failed, retried, and then skipped the same WAL.
- The new TAP test fails without the fix and passes with it for gaps of 8, 16 and 24 bytes.
- The patch is independent of the applyFlushedUpto patch and should be backported to all supported versions, including REL_14_STABLE.

**Thread state.** Srinath and João reviewed it and agree the fix is correct; João tested on master and REL_17_STABLE and calls it close to Ready for Committer. Open points are using plain RecPtr - 1 and making the test deterministic by zeroing the segment tail instead of a padding retry loop.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes the start LSN the startup process gives the walreceiver in xlogrecovery.c. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a backpatch to all supported branches; no REL_14_STABLE version is posted. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new TAP test with a WAL padding retry loop runs on every supported branch and the buildfarm. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do other callers of RequestXLogStreaming() pass an end position, and can they skip the tail of a segment in the same way?
2. Will a v2 use plain RecPtr - 1 as message #3 suggests, or is there a reason to keep the conversions in WaitForWALToBecomeAvailable()?
3. Can the test build the state directly, by truncating or zeroing the segment tail on the standby, so that the 5-attempt padding retry loop is not needed?
4. Which branches is the fix meant for, given that the CommitFest entry targets 14 and the posted patch is for master with no REL_14_STABLE version?
5. Is the new TAP test meant to be backpatched with the fix, and does it pass unchanged on each back branch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the recovery and walreceiver start-up code in xlogrecovery.c; none has spoken in the visible thread.
- **Liability:** A change to the streaming start point on every supported branch, plus a new recovery TAP test whose stability on the buildfarm the committer owns for the life of those branches.
- **Author follow-up:** The author offered to send a REL_14_STABLE version in message #1, but has not yet replied to the review comments or posted a v2.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The fix addresses a silent hang that any streaming standby can hit after a walreceiver restart, and it touches one backend file plus a test. Two reviewers agree the fix is correct; the remaining points are a simpler expression and a deterministic test.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier proposal on this idea, and the thread cites related work (message #1) that is independent of this patch, not a rejected attempt at the same fix.
- standards-compat: 0 objection(s). The patch adds no SQL syntax, GUC, catalog object, psql feature or dump content, and it does not change the replication protocol. The only behaviour change is the walreceiver's restart point when RecPtr falls exactly on a segment boundary, which corrects a hang that message #1 and message #4 reproduce. Existing users and client tools see nothing new. Backpatching it does not change any user-facing contract, and the thread has already settled the meson.build differences on back branches.
- replication-ondisk: 0 objection(s). The patch changes only the start LSN the startup process gives the walreceiver; it adds no on-disk, WAL record, catalog, protocol or ABI change, so back-patching raises no format concern. The new start point is the segment recovery is already reading, which a slot's restart_lsn or the normal retention rules keep on the primary, and the timeline still comes from tliRecPtr.
- security: 0 objection(s). The proposal changes only the streaming start point the standby requests after a walreceiver restart. It adds no privilege, role, GUC, view, error text or new user-reachable path. The start point moves back by at most one segment, and only when RecPtr is on a segment boundary, so the standby requests WAL it is already entitled to. I see no security or privilege aspect to object to.
- maintenance-fit: 0 objection(s). This is a small bug fix in core recovery code. It changes behavior only when RecPtr falls exactly on a segment boundary, adds no GUC, API, catalog or hot-path work, and two reviewers have engaged. The test's padding retry loop could be flaky on the buildfarm, but that is test design and out of scope here.
- already-exists: 0 objection(s). This is a bug fix for a standby hang. No existing feature, setting or extension prevents it. A restore_command can recover the missing tail of the segment, but it needs a WAL archive and only works around the bug, so the need is not already met.
- demand: 0 objection(s). This is a bug fix for a standby that silently stops replaying after a routine walreceiver restart, such as a primary_conninfo change or a lost connection. Every streaming standby on every supported branch can hit it. The demand question does not apply to a hang reproduced on master and REL_17_STABLE, and a reviewer called it hard to diagnose in the field (message #4).

### Precedents retrieved

- P1: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P2: [Map WAL segment files on PMEM as WAL buffers](https://commitfest.postgresql.org/patch/3181/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P3: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [Move walreceiver state assignment (to WALRCV_STREAMING) in WalReceiverMain()](https://commitfest.postgresql.org/patch/4698/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P5: [Fix possible segfault in walreceiver](https://commitfest.postgresql.org/patch/2873/) (cf, Withdrawn, 2020-12-17) - judged not relevant; reason summarised from its thread
- P6: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P7: [Persist injection points across server restarts](https://commitfest.postgresql.org/patch/5731/) (cf, Returned with feedback, 2025-07-02) - judged not relevant; reason summarised from its thread
- P8: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P9: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P10: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P11: [Simplify standby state machine a bit in WaitForWALToBecomeAvailable()](https://commitfest.postgresql.org/patch/3965/) (cf, Withdrawn, 2023-03-03) - judged not relevant
- P12: [Nonreplayable XLog records by means of overflows and >MaxAllocSize lengths](https://commitfest.postgresql.org/patch/3590/) (cf, Returned with feedback, 2022-12-02) - judged not relevant
- P13: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P14: [automatic restore point](https://commitfest.postgresql.org/patch/1788/) (cf, Rejected, 2018-09-30) - judged not relevant
- P15: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P16: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P17: [Add support for --end-segment-command in pg_receivewal](https://commitfest.postgresql.org/patch/1040/) (cf, Returned with feedback, 2017-03-21) - judged not relevant
- P18: [Revert "When WalSndCaughtUp, sleep only in WalSndWaitForWal()."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=72a3dc321d76c93842d502793f93b9dc2d2305b2) (revert, Reverted, 2020-04-25) - judged not relevant
- P19: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant
- P20: [delay starting WAL receiver](https://commitfest.postgresql.org/patch/4117/) (cf, Withdrawn, 2023-01-11) - judged not relevant
- P21: [Revert "Fix replay of create database records on standby"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=bf902c13930c268388644100663f2998868b6e85) (revert, Reverted, 2022-03-29) - judged not relevant
- P22: [Start WAL receiver eagerly](https://commitfest.postgresql.org/patch/3376/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P23: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P24: [CREATE SEQUENCE with RESTART option](https://commitfest.postgresql.org/patch/3069/) (cf, Rejected, 2021-07-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 51 s; cost reported by the CLI: $0.70
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 43 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:51.250Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
