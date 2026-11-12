# Design pre-review: Release replication slot on error in SQL-callable slot functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7097](https://commitfest.postgresql.org/patch/7097/) (Needs review); authors: Bharath Rupireddy (bharathrupireddy), SATYANARAYANA NARLAPURAM (satyanarayanan)
- Thread: [\[PATCH\] Release replication slot on error in SQL-callable slot functions](https://www.postgresql.org/message-id/flat/CAHg%2BQDeuf9tCq3ce%3DkgFMJP0m%3DPZC%2Bwi6B%3DyS%2B7V0vNXjLS31w%40mail.gmail.com), started 2026-05-09
- Messages read: 90 of 90
- Patch prose read: v5-0001-Release-replication-slot-on-error-in-slot-SQL-functions.patch, v1-0001-Release-replication-slot-on-error-in-slot-SQL-functions.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is reproducible with plain SQL and the goal is not in dispute. One fixable level 2 objection survived, standards-compat-1: releasing the slot on subtransaction abort changes what extensions see in minor releases back to PG14, and the thread reviewed only the commit side. That is a back-patch risk judgement for a committer who knows the slot code. It does not stop the idea.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The material shows commit messages for v1 and v5 only, so the v19 callback design and its claim that release cannot raise an error during subtransaction abort were not checked. The thread does not say whether v19 was committed.

## The proposal, restated

**Problem.** SQL slot functions set MyReplicationSlot and can error before releasing the slot. If a PL/pgSQL (or PL/Perl, PL/Python, PL/Tcl) exception block catches the error, the top-level cleanup never runs. The next slot call hits an assertion, or in release builds the slot stays active, holding back WAL removal and vacuum.

**Proposed design.** The first versions wrapped each slot function in PG_TRY/PG_CATCH. From v7 on, the patch records the subtransaction id that acquired the slot and releases the slot in a new subtransaction-end callback when that subtransaction aborts. Temporary slots are not dropped on a caught error, and nothing happens on subtransaction commit.

**User-visible surface.**
- Documentation change: temporary slots are dropped on errors, but not on an error caught in a subtransaction such as a PL/pgSQL exception block.
- A slot acquired in a subtransaction is now released when that subtransaction aborts, which changes behavior for extensions that hold slots.

**What the author claims.**
- The bug exists on all supported branches back to PG14 and is reachable with plain SQL.
- Affected functions include slot create, copy, advance, drop, and the get/peek changes functions.
- pg_sync_replication_slots() is not affected because it already uses PG_ENSURE_ERROR_CLEANUP().
- Releasing the slot during subtransaction abort cannot raise an error (Sawada's reading of the code).
- The fix is meant for back-patching to all supported branches.

**Thread state.** Many reviewers (Sawada, Shveta, Amit, Hou, Ashutosh, Horiguchi) shaped v19, and Sawada said in message #86 he planned to push. Leftover temporary slots after a failed create were moved to a separate thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-05-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Release on subtransaction abort changes extension-visible slot state in minor releases

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

An extension acquires a slot inside BeginInternalSubTransaction(), catches an error with PG_CATCH, calls RollbackAndReleaseCurrentSubTransaction() and then calls ReplicationSlotRelease(). After the minor update the rollback has already cleared MyReplicationSlot, so the extension's release call fails an assertion or dereferences a NULL slot. The user finds out when the backend crashes on the first decoding error after the update.

The thread studied the commit side for extensions in detail and removed the handoff and WARNING to avoid minor-release changes (messages #70, #73, #81). The abort side gets no such review, but it is the part that now runs inside AbortSubTransaction() on every supported branch back to PG14. Existing extension cleanup code was written for a world where slot state survives a subtransaction rollback. The v19 commit message is not in the material, so it is unknown whether release notes will warn extension authors.

Thread: [#70 (Bharath Rupireddy, 2026-09-04)](https://www.postgresql.org/message-id/CALj2ACUVTXdtGSdH7ez7anCB1cT8o78KB9RJ5c1uutRd0OsgsQ%40mail.gmail.com), [#73 (Masahiko Sawada, 2026-09-16)](https://www.postgresql.org/message-id/CAD21AoB-dkBCo06isKxeF-W4Vah8YnoGVbOBOs1SysoSQRr45w%40mail.gmail.com), [#81 (Bharath Rupireddy, 2026-09-17)](https://www.postgresql.org/message-id/CALj2ACUBJ53hY8EOUJnWA7m1GzBP_CoPo9N6e01x5%2Bq9GPQoXQ%40mail.gmail.com), [#86 (Masahiko Sawada, 2026-09-18)](https://www.postgresql.org/message-id/CAD21AoDT_ypWw%2BGpQ377Dat4w%3Dj7iqhqC1m-X2XYDxtXjYeBtA%40mail.gmail.com)

**Question:** Was extension code that releases the slot after rolling back its own subtransaction checked, and will the back-branch commit message or release notes tell extension authors about the new abort behavior?

Counter-check: held. The thread reviewed the commit path for extensions (#70, #73) but not an extension that calls ReplicationSlotRelease() after rolling back its own subtransaction. Amit's remark in #58 accepts the new restriction without covering this crash on back branches.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes when a backend releases a replication slot, which affects WAL retention by an active slot. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | A slot that an extension acquires in a subtransaction is now released when that subtransaction aborts. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The fix is meant for all supported branches back to PG14 and changes slot behavior on subtransaction abort there. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a subtransaction-end callback and a recorded subtransaction id that slot code must keep correct. |
| performance on the hot path | **yes** | The new callback runs at every subtransaction end, and the review notes put the cost at one variable comparison. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Was extension code that calls ReplicationSlotRelease() after rolling back its own subtransaction checked against the new abort behavior?
2. Will the back-branch commit message or release notes tell extension authors that a subtransaction abort now releases the slot?
3. Should the abort-time release go to all branches back to PG14, or should back branches get a narrower fix?
4. How does a slot that an extension holds across transactions behave after this change?
5. Why is a subtransaction-end callback preferred over ResourceOwner, as message #55 suggested, at least on HEAD?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Sawada reviewed the design and said in message #86 that he planned to push. The owner needs to know the replication slot code and subtransaction abort processing.
- **Liability:** The owner signs for slot release inside AbortSubTransaction() on every supported branch back to PG14, including any extension breakage that shows up after a minor release.
- **Author follow-up:** The patch went through 19 versions over about 90 messages with many reviewers, which suggests the authors stay engaged.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Only sessions that catch a slot function error in a subtransaction hit the bug, but the result is an assertion failure or a slot stuck active that holds back WAL removal and vacuum. The v19 patch was not read; early versions are small and touch slot code only.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-branch release on subxact abort changes behavior for extensions. Refuted (not specific): Duplicate of standards-compat-1, which gives the more concrete scenario for the same back-branch abort behavior change.
- `maintenance-fit-1` Subxact-end callback chosen over ResourceOwner without a stated reason. Refuted (no failure scenario): The scenario is today's behavior, and messages #73 and #80 left it unchanged on purpose to keep the back-branch fix small. A subtransaction-end callback copies existing practice such as AtEOSubXact_LargeObject() and AtEOSubXact_Files(), so item 12 does not fail.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at this idea. The thread mentions no earlier discussion of it either, and the only related thread it names is a split-off for temporary slots whose creation fails (message #90).
- standards-compat: 1 objection(s). The SQL-level contract does not change. No syntax, GUC, catalog or pg_dump change is involved, and the documentation hunk describes what already happens to temporary slots.
- replication-ondisk: 1 objection(s). The patch adds no on-disk, WAL, catalog or protocol change. The release runs only in-process memory and existing slot code during subxact abort, and message #56 argues it cannot raise an error there. Leftover temporary slots after a failed create are a pre-existing issue and were moved to a separate thread.
- security: 0 objection(s). The fix adds no new privilege, function, or exposure; the slot functions keep requiring superuser or the REPLICATION attribute. Releasing the slot on subtransaction abort removes a session-local leak that held back WAL removal and vacuum. Temporary slots left behind after a failed create already behaved this way and were moved to a separate thread.
- maintenance-fit: 1 objection(s). Apart from the ResourceOwner question, maintenance and fit look sound. The fix is a backpatchable bug fix in core code, several committers in this area reviewed it, and the per-subxact cost is one variable comparison.
- already-exists: 0 objection(s). This is a crash and resource-leak fix with no existing alternative. Users cannot avoid it short of not catching errors, and PG_ENSURE_ERROR_CLEANUP (used by pg_sync_replication_slots()) would need per-function wrapping, which the thread weighed and dropped. Horiguchi's ResourceOwner suggestion in message #55 got no reply, but an AtEOSubXact callback follows the existing practice of AtEOSubXact_LargeObject() and AtEOSubXact_Files(), so I see no concrete failure scenario to raise from this angle.
- demand: 0 objection(s). This is a bug fix, so the demand bar is low. Plain SQL in a PL/pgSQL exception block can crash an assert build, and in a release build it leaves a slot active that holds back WAL removal and vacuum. Several reviewers reproduced it independently (messages #1, #3, #5, #7, #74), and there is no workaround short of ending the session. The cost falls on the subtransaction abort path only when a slot is held.

### Precedents retrieved

- P1: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P2: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P3: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P4: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P5: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P6: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P7: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P8: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant; reason summarised from its thread
- P9: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant; reason summarised from its thread
- P10: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant; reason summarised from its thread
- P11: [enable pg_recvlogical to use temporary slots](https://commitfest.postgresql.org/patch/5363/) (cf, Returned with feedback, 2025-01-27) - judged not relevant
- P12: [Detect escape of ErrorContextCallback stack pointers (and from PG_TRY() )](https://commitfest.postgresql.org/patch/2722/) (cf, Withdrawn, 2020-09-10) - judged not relevant
- P13: [Fix off-by-one in logical slot resource retention](https://commitfest.postgresql.org/patch/1084/) (cf, Returned with feedback, 2017-10-01) - judged not relevant
- P14: [Fix assertion failure in SnapBuildInitialSnapshot()](https://commitfest.postgresql.org/patch/4149/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P15: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P16: [rich constructor for plpy.Error class, possible raise structured exception](https://commitfest.postgresql.org/patch/387/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P17: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant
- P18: [Rename trim_array() for consistency with the rest of array_* functions](https://commitfest.postgresql.org/patch/5335/) (cf, Withdrawn, 2024-11-04) - judged not relevant
- P19: [Surround CheckRelation\[Oid\]LockedByMe() with USE_ASSERT_CHECKING](https://commitfest.postgresql.org/patch/5096/) (cf, Withdrawn, 2024-07-01) - judged not relevant
- P20: [Implement ASSERTIONs per the SQL standard.](https://commitfest.postgresql.org/patch/1604/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P21: [Upper limit arguments of pg_logical_slot_xxx_changes functions accept invalid values](https://commitfest.postgresql.org/patch/1725/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P22: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P23: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant
- P24: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 98 s; cost reported by the CLI: $4.63
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 60 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:19:03.450Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
