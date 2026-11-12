# Design pre-review: [Patch]Incorrect usage of PageGetFreeSpace instead of PageGetHeapFreeSpace in heap_xlog_visible

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6712](https://commitfest.postgresql.org/patch/6712/) (Needs review); authors: Alexey Makhmutov (lerm)
- Thread: [\[Patch\]\[Bug\] Incorrect usage of PageGetFreeSpace instead of PageGetHeapFreeSpace in heap_xlog_visible](https://www.postgresql.org/message-id/flat/66c011a7-2f26-4ab9-a5b0-1afcb8523367%40postgrespro.ru), started 2026-04-23
- Messages read: 1 of 1
- Patch prose read: 0001-Use-PageGetHeapFreeSpace-in-heap_xlog_visible.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The patch replaces one function call in heap_xlog_visible so that standby redo records the same FSM value the primary does, and PG 19 already has the fix per message #1. The remaining points are back-patch details: the branch range, the PG 18 variant, and the rebase CFbot asks for. Confidence is medium because only one message is visible and no code was read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that heap_xlog_visible is the only heap caller of PageGetFreeSpace is unchecked. The PG 18 variant and the cost of the extra line-pointer check during redo could not be judged.

## The proposal, restated

**Problem.** On a standby, heap_xlog_visible computes FSM free space with PageGetFreeSpace, which ignores the MaxHeapTuplesPerPage limit. A heap page whose line pointers are all used, for example by many HOT redirects, is recorded in the replica FSM as having free space. Users of PG 14 to 18 replicas are affected; the author says PG 19 already fixed this in a881cc9.

**Proposed design.** The patch replaces PageGetFreeSpace with PageGetHeapFreeSpace in heap_xlog_visible, so redo reports 0 free space when no new tuple fits. The patch targets PG 14 to 17. PG 18 needs an adjusted version because the function moved to heapam_xlog.c, and the author points to a version in the earlier thread.

**What the author claims.**
- The bug is an oversight in commit ab7dbd6.
- heap_xlog_visible is the only place that uses PageGetFreeSpace for heap pages.
- The patch is already fixed in PG 19 by a881cc9 and is not relevant there.
- In the test case, without the fix, the master shows 0 free space and the replica about 1150 bytes; with the fix, both show 0.
- The mismatch is rarely seen in practice.

**Thread state.** Only the first message is visible, and nobody has replied. CFbot reports that the patch needs a rebase, and the entry moved to a later CommitFest without review.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes the free space value that heap_xlog_visible writes to the standby FSM during redo. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix targets back branches only, PG 14 to 17, with a separate variant needed for PG 18. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Should the fix go to all branches from 14 to 18, and can a PG 18 patch be attached to this thread?
2. Against which branch does the attached patch apply, given that CFbot reports it needs a rebase?
3. What does an inserter on a promoted standby do with the wrong FSM entry, and what does that cost in practice?
4. Can the shell test case become a TAP test, or is the fix meant to go in without a test?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows heap redo and the free space map would have to own it; nobody has replied in the visible thread.
- **Liability:** A one-call change in heap redo on five back branches, shipped in minor releases. The FSM value is a hint, so the risk is small, but a back-patched redo change stays with the committer.
- **Author follow-up:** No sign either way; the author posted once, split this from an earlier thread, and has not yet posted a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Message #1 calls the mismatch rare, and it affects only standbys with heap pages at the MaxHeapTuplesPerPage limit. The patch is a one-call change in heapam.c, but nobody has replied and the entry moved CommitFests without review.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list. Message #1 says this spins off an earlier thread on a related second issue and that PG 19 already fixed it in a881cc9; the material reports no rejection there. A one-line back-branch bug fix gives no prior-rejection reason to apply.
- standards-compat: 0 objection(s). Nothing objectionable from the standards and compatibility angle. The fix changes only the free space value that redo writes to the standby FSM. It adds no syntax, GUC, catalog object, psql behaviour or pg_dump content, and it does not change what clients see. The back-branch FSM value is a hint, and redo on unpatched standbys still behaves as it does today. The angle cannot check whether the FSM value changes plans or other behaviour on existing branches, because the code is not shown.
- replication-ondisk: 0 objection(s). The fix changes only the value that redo writes to the standby FSM, and the FSM is advisory, so inserters recheck the page. It adds no WAL record, on-disk format, catalog or ABI change, so it is safe to backpatch and to run in mixed minor versions. Stale FSM entries left on existing standbys correct themselves on later updates.
- security: 0 objection(s). The patch changes how WAL redo on a standby computes the free space it records in the FSM for a heap page. It adds no privilege, role, setting, or interface, and it exposes no data to any user, so there is nothing to object to from the security angle. A stale FSM entry only costs an inserter a recheck, and the patch makes that less likely.
- maintenance-fit: 0 objection(s). No objection from the maintenance angle. The patch swaps one function call in heap_xlog_visible to match what every other heap code path already does, adds no state, API or mechanism, and only costs a line-pointer check during redo of visibility records. The PG 18 variant and the branch range are back-patch details for the committer to settle.
- already-exists: 0 objection(s). This is a back-branch bug fix to an existing redo path. No existing feature, extension or tool gives standby redo the correct FSM value, so there is nothing to point to as an alternative. The fix uses the existing PageGetHeapFreeSpace and adds no new mechanism.
- demand: 0 objection(s). This is a one-line bug fix that makes back branches match the heap free-space rule already used on master. A demand test does not apply, and the cost falls on nobody. Low field impact (message #1 calls the mismatch rare) is a question of back-patch priority, which other panel members can weigh better than the demand angle.

### Precedents retrieved

- P1: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P2: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P3: [Revert "Avoid creation of the free space map for small heap relations."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a23676503b746b7f1588cd2ab0c60411032d32da) (revert, Reverted, 2019-01-28) - judged not relevant
- P4: [Better tracking of free space during SP-GiST index build](https://commitfest.postgresql.org/patch/739/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P5: [Auto-fill the free space map by scanning the buffer cache or by checking pages written by the background writer](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Vacuum) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P6: [Skip all-visible pages during second HeapScan of CIC](https://commitfest.postgresql.org/patch/1057/) (cf, Returned with feedback, 2017-03-24) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P8: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant; reason summarised from its thread
- P9: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant; reason summarised from its thread
- P10: [Adding TRIM_SPACE option to COPY](https://commitfest.postgresql.org/patch/6289/) (cf, Withdrawn, 2026-04-13) - judged not relevant; reason summarised from its thread
- P11: [Pruning never visible changes](https://commitfest.postgresql.org/patch/3900/) (cf, Returned with feedback, 2022-10-27) - judged not relevant; reason summarised from its thread
- P12: [COPY FREEZE and PD_ALL_VISIBLE](https://commitfest.postgresql.org/patch/401/) (cf, Returned with feedback, 2015-12-24) - judged not relevant; reason summarised from its thread
- P13: [Document spaces in .pgpass need to be escaped](https://commitfest.postgresql.org/patch/3335/) (cf, Withdrawn, 2021-09-30) - judged not relevant
- P14: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P15: [Adding CommandID to heap xlog records](https://commitfest.postgresql.org/patch/3882/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P16: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P17: [Avoid possible null pointer dereference](https://commitfest.postgresql.org/patch/5506/) (cf, Withdrawn, 2025-02-06) - judged not relevant
- P18: [Client Connection redirection support for PostgreSQL](https://commitfest.postgresql.org/patch/1673/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P19: [In bitmap scan explain node show how many relevant heap pages were not examined](https://commitfest.postgresql.org/patch/2443/) (cf, Returned with feedback, 2020-03-31) - judged not relevant
- P20: [XLog size reductions: Reduced XLog record header size](https://commitfest.postgresql.org/patch/4386/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P21: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - judged not relevant
- P22: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant
- P23: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant
- P24: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 72 s; cost reported by the CLI: $0.64
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 42 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:01:31.691Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
