# Design pre-review: Temporary slot leak when creation fails in a subtransaction

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7347](https://commitfest.postgresql.org/patch/7347/) (Needs review); authors: Bharath Rupireddy (bharathrupireddy)
- Thread: [Temporary slot leak when creation fails in a subtransaction](https://www.postgresql.org/message-id/flat/CALj2ACXDsQhq0MFuyRfZ0BJT6F93CNQmG5iv%2BwDzKp%2BYyOGD%2BA%40mail.gmail.com), started 2026-09-25
- Messages read: 7 of 7
- Patch prose read: v1-0001-Drop-a-temporary-replication-slot-if-its-creation.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is confirmed in message #2 and the fix changes nothing on disk or in shared memory. The one surviving objection, already-exists-1, is a fixable design point: on HEAD the backend-local variable would sit beside RS_EPHEMERAL, and the author's reason against ephemeral-first creation applies to back branches only. That choice, and a back-patch to 14, need a committer who knows the slot code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, two points are unchecked: whether a slot whose creation finished before a later error in the same subtransaction keeps its documented behaviour, and whether walsender slot creation outside a transaction is unaffected. Only 3 of 7 messages were shown.

## The proposal, restated

**Problem.** A temporary logical or physical slot whose creation fails inside a subtransaction, with the error caught by a PL/pgSQL exception block, stays until the session ends. A leftover logical slot holds WAL and catalog_xmin, and has no confirmed_flush_lsn, so decoding from it can return a partial transaction. A leftover physical slot holds WAL.

**Proposed design.** A backend-local variable marks a slot as being created and is reset when the slot is released. If the subtransaction that created a temporary slot aborts with the variable still set, the backend drops the slot instead of releasing it. Nothing changes on disk or in shared memory, so the fix can be back-patched to 14.

**User-visible surface.**
- Behaviour change: a temporary slot is dropped when an error during its creation is caught in a subtransaction.
- Documentation of pg_create_physical_replication_slot, pg_create_logical_replication_slot, CREATE_REPLICATION_SLOT and the temporary column of pg_replication_slots says so.

**What the author claims.**
- The bug exists on HEAD and all supported branches back to 14.
- The fix is back-patchable and covers both logical and physical temporary slots.
- Without the fix, the test shows the leftover slot returning a transaction with one of its two inserts.
- Ephemeral-first creation, a new slot state, or a new ReplicationSlot field are invasive or break ABI, so they are not back-patchable.
- A resource owner approach does not fit: slots can outlive their (sub)transaction and the walsender creates slots without a transaction.

**Thread state.** Kuroda confirmed the bug, and shveta malik and Narayanan Venkateswaran plan to review a patch that applies only on top of v19 of another thread's patch. Open: shveta asks whether HEAD should create slots as ephemeral first, keeping the variable for back branches, and the leak of failed persistent physical slots is not yet handled.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Second "creation in progress" mechanism next to the existing ephemeral state

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A later patch adds a slot creation path, such as a new sync or creation function, and uses only one of the two mechanisms. Nothing fails at once. The leak comes back in that path and is found only when someone hits a caught error during creation.

Postgres already has RS_EPHEMERAL, which drops a persistent logical slot on release if its creation fails. The patch adds MyReplicationSlotCreating for temporary slots, and message #6 proposes extending it to persistent physical slots, so two mechanisms would do the same job on HEAD. The author calls ephemeral-first invasive and not back-patchable, but that argument covers back branches only. In message #7, shveta asks for the ephemeral state on HEAD, and the thread has not settled this.

Thread: [#1 (Bharath Rupireddy, 2026-09-25)](https://www.postgresql.org/message-id/CALj2ACXDsQhq0MFuyRfZ0BJT6F93CNQmG5iv%2BwDzKp%2BYyOGD%2BA%40mail.gmail.com), [#6 (Bharath Rupireddy, 2026-09-30)](https://www.postgresql.org/message-id/CALj2ACV%2BOzVikz4dwgWi399AHas3RneVsWfy-sF43bCf65CprA%40mail.gmail.com), [#7 (shveta malik, 2026-09-30)](https://www.postgresql.org/message-id/CAJpy0uAnNgCaOpQcum2uqdq-4in_k7cQc0vqzVbyufeMfpwGCA%40mail.gmail.com)

**Question:** On HEAD, what stops temporary and physical slots from starting as RS_EPHEMERAL, so that the backend-local variable is needed only on back branches?

Counter-check: held. The author's reason against ephemeral-first creation is that it cannot be back-patched, which does not apply to HEAD. Message #7 asks for ephemeral-first on HEAD and the thread has no answer yet, so the consistency question is open.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes when a temporary replication slot is dropped, in src/backend/replication/slot.c. |
| wire protocol | **yes** | protocol.sgml changes the documented behaviour of CREATE_REPLICATION_SLOT ... TEMPORARY; no message format changes. |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit message asks for a back-patch to 14, and the patch applies only on top of an unmerged patch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | HEAD would carry a backend-local creation variable beside the existing RS_EPHEMERAL state. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. On HEAD, what stops temporary and physical slots from starting as RS_EPHEMERAL, so that the backend-local variable is needed only on back branches?
2. In which order are this patch and the v19 patch from the other thread to be committed, on HEAD and on each back branch?
3. Is the CreateDecodingContext() check for an invalid confirmed_flush part of this patch, and on which branches?
4. How does the patch treat a walsender that creates a temporary slot outside a transaction?
5. Will the same variable also cover failed creation of a persistent physical slot, or is that a separate patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows replication slot lifetime, logical decoding and subtransaction abort handling. The visible messages show reviewers, and no such committer speaking.
- **Liability:** A change to slot cleanup during subtransaction abort on six branches, 14 to HEAD, for the life of each branch. A wrong drop or a missed drop holds WAL and catalog_xmin, or lets decoding return a partial transaction.
- **Author follow-up:** The author moved the discussion from the earlier thread and posted a patch with tests and documentation. A rebase was requested in message #3; the visible messages do not show the reply.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The trigger is narrow, a failed temporary slot creation caught in a subtransaction, but the result is retained WAL and partial decoding, and the fix has no cost outside that path. Review needs care in slot.c and postgres.c on six branches, and the patch needs a rebase.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` HEAD fix differs from the approach the earlier thread preferred. Refuted (not specific): Duplicate of already-exists-1. Its failure scenario is also weak: the commit message says the variable is set when the slot is created and reset on release, so a new caller of the common creation routine would get it without extra code.
- `replication-ondisk-1` Slot drop in subtransaction abort can fail in the disk-full case. Refuted (no failure scenario): Dropping a slot renames and removes its directory, which needs no free space, and the existing drop code reports a failed rename of a non-persistent slot as a warning. Top-level error recovery already drops temporary slots under the same disk-full condition.
- `maintenance-fit-1` Second "slot in creation" mechanism next to the existing ephemeral state. Refuted (not specific): Duplicate of already-exists-1, which states the same point about two mechanisms on HEAD more directly.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the list. The only earlier discussion is the predecessor thread cited in messages #1 and #7, which reports agreement on the bug and a preference for ephemeral-first on master.
- standards-compat: 0 objection(s). The only user-visible change is that a temporary slot whose creation fails inside a caught subtransaction is now dropped. No caller can rely on a half-created slot, and there is no new syntax, GUC, catalog or pg_dump state, so back-patching this does not break any compatibility contract. Two points could not be checked without the code: that a slot whose creation completed before a later error in the same subtransaction keeps its documented behaviour, and that walsender creation outside a transaction is unaffected.
- replication-ondisk: 1 objection(s). The fix adds no on-disk, WAL, catalog, protocol or ABI change, so pg_upgrade, standbys and extensions are unaffected. The persistent physical slot leak in message #6 is a separate, pre-existing bug and is not counted against this proposal.
- security: 0 objection(s). The fix has no privilege or exposure aspect. Creating a slot already needs the REPLICATION attribute, and the change only drops the backend's own temporary slot during subtransaction abort, so no role gains new access to data, WAL or other sessions. It also removes a leftover slot that held WAL, so it lowers disk-use risk.
- maintenance-fit: 1 objection(s). This is a back-patchable bugfix with no hot-path, on-disk or ABI cost. Only the long-run cost of a second creation-state mechanism on HEAD is raised.
- already-exists: 1 objection(s). This is a bug fix. Postgres has no feature or extension that drops a temporary slot when its creation fails in a caught subtransaction, so only the mechanism duplication applies from this angle.
- demand: 0 objection(s). This is a bug fix: two people reported it, Kuroda confirmed it in message #2, and a test shows a partial transaction being decoded. The trigger is narrow, but the result is WAL and catalog_xmin retention plus data loss, and the cost falls only on the failed-creation path.

### Precedents retrieved

- P1: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P2: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P3: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P4: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P5: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P6: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P7: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P8: [enable pg_recvlogical to use temporary slots](https://commitfest.postgresql.org/patch/5363/) (cf, Returned with feedback, 2025-01-27) - judged not relevant; reason summarised from its thread
- P9: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P10: [Skip replicating the tables specified in except table option](https://commitfest.postgresql.org/patch/3646/) (cf, Withdrawn, 2024-01-09) - judged not relevant; reason summarised from its thread
- P11: [Show subtransaction overflow status](https://commitfest.postgresql.org/patch/3444/) (cf, Rejected, 2022-03-22) - judged not relevant
- P12: [Revert "Add test case for obsoleting slot with active walsender"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96795176810b979a2bf1f4563bdcb161679d5b95) (revert, Reverted, 2021-06-20) - judged not relevant
- P13: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P14: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P15: [Prepare Transaction support for ON COMMIT DROP temporary tables](https://commitfest.postgresql.org/patch/1954/) (cf, Returned with feedback, 2019-02-01) - judged not relevant
- P16: [Automatic HASH and LIST partition creation](https://commitfest.postgresql.org/patch/2694/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P17: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P18: [Revert "Forbid DROP SCHEMA on temporary namespaces"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65192e02441cedd106b6abebe0036fb8cc124fb3) (revert, Reverted, 2020-01-08) - judged not relevant
- P19: [Flush some statistics within running transactions](https://commitfest.postgresql.org/patch/6389/) (cf, Returned with feedback, 2026-03-25) - judged not relevant
- P20: [Revert "Drop unnamed portal immediately after execution to completion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=910690415b661186ae44e3b5e538e23eaa48de1b) (revert, Reverted, 2025-11-14) - judged not relevant
- P21: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P22: [Be strict when request to flush past end of WAL in WaitXLogInsertionsToFinish](https://commitfest.postgresql.org/patch/7294/) (cf, Withdrawn, 2026-09-12) - judged not relevant
- P23: [Fix database creation during installchecks for ICU cluster](https://commitfest.postgresql.org/patch/3975/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P24: [Fix creation of empty .log files during log rotation](https://commitfest.postgresql.org/patch/5365/) (cf, Rejected, 2026-01-08) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 116 s; cost reported by the CLI: $1.21
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:42:13.028Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
