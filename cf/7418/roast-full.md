# Design pre-review: Remove redundant MultiXactIdIsRunning() check in HeapTupleSatisfiesUpdate()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7418](https://commitfest.postgresql.org/patch/7418/) (Needs review); authors: Zhao Song (pg.zs)
- Thread: [Remove redundant MultiXactIdIsRunning() check in HeapTupleSatisfiesUpdate()](https://www.postgresql.org/message-id/flat/7201E833-0100-4BDE-9161-45B9FFA4F458%40icloud.com), started 2026-10-08
- Messages read: 4 of 4
- Patch prose read: v2-0001-Remove-redundant-MultiXactIdIsRunning-check-in-He.patch, v1-0001-Remove-redundant-MultiXactIdIsRunning-check-in-He.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because the retained comment states the dependency on the first check and multixact membership is a documented invariant. The patch removes a branch that message #1 argues cannot be reached, and one reviewer agreed in message #3. Confidence is medium because the code was not read and no committer has confirmed the reasoning.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claim that nothing between the two checks can change the result rests on message #1 and message #3. The lock held across both checks could not be checked.

## The proposal, restated

**Problem.** In the HEAP_XMAX_IS_MULTI branch of HeapTupleSatisfiesUpdate(), a second MultiXactIdIsRunning() call follows an earlier one that already returns TM_BeingModified if any member runs. The author says the second call always returns false, so its else branch is dead code. This costs a multixact member lookup and a proc array scan when the updater aborted, and it makes the code harder to read.

**Proposed design.** The patch removes the second MultiXactIdIsRunning() call and its unreachable TM_BeingModified branch. When the updater aborted, the code sets the HEAP_XMAX_INVALID hint bit and returns TM_Ok. v2 drops the "By here ..." comment and keeps the "There's no member ..." comment, as the reviewer asked.

**What the author claims.**
- Multixact membership is immutable, and a member that was not running cannot become running again, so the second check always returns false.
- Commit 05315498012 made the check redundant when it changed the first check back to cover all members.
- The patch does not change behavior.
- The patch avoids a multixact member lookup and a proc array scan when the updater aborted.

**Thread state.** Chao Li agreed with the analysis and asked for a comment change, which v2 makes. No committer has replied, and the CommitFest entry is in Needs review.

## 2. Prior attempts

### Remove redundant NULL check in clause_selectivity_ext() (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/5188/](https://commitfest.postgresql.org/patch/5188/)
- **Thread:** [https://www.postgresql.org/message-id/flat/016e33b7-2830-4300-bc89-e7ce9e613bad@tantorlabs.com](https://www.postgresql.org/message-id/flat/016e33b7-2830-4300-bc89-e7ce9e613bad@tantorlabs.com)
- **Date:** decided 2024-09-30; started 2024-08-19
- **Outcome:** Rejected (cf)
- **Who objected:** Tom Lane, David Rowley (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane wanted the check kept because it is cheap and extensions call clauselist_selectivity and related functions and may rely on it not crashing on a NULL. He also said the list_length check proves nothing about NULL list elements, and that removing such checks one at a time without agreed invariants is not wise. David Rowley agreed, saying no speedup could be measured and no benchmark was given, and suggested marking the entry rejected.
- **Reason (quote):** "Certainly, such an assumption could be argued to be a bug they should fix; but I'm disinclined to make them jump through that hoop for a vanishingly small performance improvement." ([message](https://www.postgresql.org/message-id/1373706.1724079757%40sss.pgh.pa.us))
- **What is different this time:** P2 removed a defensive NULL check in a function that extensions call, so outside callers could depend on it. This patch removes a branch inside HeapTupleSatisfiesUpdate() that no caller can reach, because multixact membership is immutable.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch edits HeapTupleSatisfiesUpdate() in heapam_visibility.c, where setting HEAP_XMAX_INVALID now relies on the first MultiXactIdIsRunning() check alone. |
| performance on the hot path | **yes** | The patch removes a multixact member lookup and proc array scan from the aborted-updater path of a tuple visibility routine. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do both MultiXactIdIsRunning() calls run under the same buffer lock, and does the argument depend on that?
2. Do other visibility routines, such as HeapTupleSatisfiesMVCC, have the same double check, and should this patch cover them?
3. Is there a measurement of the saved member lookup and proc array scan, or is the patch justified by readability alone?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows heap tuple visibility and multixact handling would have to own it. No committer has replied in the visible thread.
- **Liability:** The committer signs for the claim that the removed branch is unreachable. If that is wrong, an aborted updater with running lockers gets HEAP_XMAX_INVALID and the row locks are lost.
- **Author follow-up:** The author posted v2 after the review comment in message #3, which made the requested comment change.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 2 KiB in one file, tagged Refactoring Only, with no user-visible change and no measured gain. It needs a committer who knows multixact visibility to confirm the reasoning, and none has replied.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Removed check leaves lockers unprotected if the first check narrows again. Refuted (no failure scenario): The scenario needs a later patch to undo commit 05315498012 a few lines above the retained "There's no member ..." comment, which states the dependency. [P2] concerned extension callers of an exported function without agreed invariants, but multixact membership is a documented invariant.

### Review angles

- prior-attempts: 0 objection(s). The one similar precedent [P2] died because extensions might pass NULL and no speedup was shown. That does not apply here: no caller can observe the removed branch, and the patch is mainly justified by readability. The missing measurement is a weak point but not a reason to reject; the thread reports no earlier discussion beyond the commit history in message #1.
- standards-compat: 0 objection(s). The patch removes dead code inside HeapTupleSatisfiesUpdate() and changes no SQL syntax, GUC, catalog, pg_dump output, psql behavior or client-visible result. [P2] does not apply: its concern was extensions passing NULL to an exported function, and this patch keeps the function signature and its results unchanged.
- replication-ondisk: 0 objection(s). The patch changes no on-disk format, WAL record, catalog, wire protocol or extension API. The only persistent effect is the HEAP_XMAX_INVALID hint bit, which is still set only after MultiXactIdIsRunning() has found no running member; it goes through the same SetHintBits() path, so checksum and standby behavior do not change.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It removes a redundant visibility check inside HeapTupleSatisfiesUpdate() and adds no SQL surface, privilege, setting or new information path, so no unprivileged user gains any ability or knowledge. Whether the branch is truly unreachable is a correctness question for other angles.
- maintenance-fit: 1 objection(s). From the maintenance angle the change is small and local to one file, has no API, catalog or hot-path cost, and has nothing for an extension to depend on. The one concern above can be fixed with an assertion.
- already-exists: 0 objection(s). From the already-exists angle there is nothing to raise. The patch removes dead code inside one visibility routine and adds no feature, so no existing feature, extension or hook could already meet the need, and it adds no second mechanism. P2 is about removing a defensive check that extensions might rely on, and that reason does not apply to an internal branch shown unreachable.
- demand: 0 objection(s). The demand angle does not apply well here. The patch removes an unreachable branch in one internal function, adds no user-visible surface, and costs non-users nothing. Its main gain is readability, which needs no field demand. The speedup on the aborted-updater path is unmeasured, but the patch does not depend on it. [P2] does not fit: there, extensions could rely on the removed check, and no outside caller can reach this branch. Whether the branch is truly unreachable is a safety question for another angle.

### Precedents retrieved

- P1: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P2: [Remove redundant NULL check in clause_selectivity_ext()](https://commitfest.postgresql.org/patch/5188/) (cf, Rejected, 2024-09-30) - used in the review; reason summarised from its thread
- P3: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P4: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P5: [Remove redundant DISTINCT when GROUP BY already guarantees uniqueness](https://commitfest.postgresql.org/patch/6869/) (cf, Withdrawn, 2026-07-23) - judged not relevant; reason summarised from its thread
- P6: [Revert the addition of GetMaxBackends() and related stuff.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7fc0e7de9fb8306e84d1c15211aba4308f694455) (revert, Reverted, 2022-04-12) - judged not relevant
- P7: [Avoid hiding shared filesets in pg_ls_tmpdir (pg_ls_* functions for showing metadata ...)](https://commitfest.postgresql.org/patch/2377/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P8: [Align verify_heapam.c error message offset with test expectations](https://commitfest.postgresql.org/patch/6423/) (cf, Withdrawn, 2026-05-12) - judged not relevant; reason summarised from its thread
- P9: [Add pg_wait_for_lockers() function](https://commitfest.postgresql.org/patch/4480/) (cf, Withdrawn, 2024-07-22) - judged not relevant; reason summarised from its thread
- P10: [Refactoring of heapam code](https://commitfest.postgresql.org/patch/700/) (cf, Returned with feedback, 2016-09-26) - judged not relevant; reason summarised from its thread
- P11: [Allow tuple visibility checks without hint-bit maintenance](https://commitfest.postgresql.org/patch/7212/) (cf, Withdrawn, 2026-08-30) - judged not relevant
- P12: [Redundant qualifier elimination](https://commitfest.postgresql.org/patch/7127/) (cf, Returned with feedback, 2026-08-14) - judged not relevant
- P13: [Pruning never visible changes](https://commitfest.postgresql.org/patch/3900/) (cf, Returned with feedback, 2022-10-27) - judged not relevant
- P14: [Refactor how we form HeapTuples for CatalogTuple(Insert|Update)](https://commitfest.postgresql.org/patch/6221/) (cf, Withdrawn, 2026-01-30) - judged not relevant
- P15: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P16: [Allow tuple visibility checks without hint-bit, maintenance](https://commitfest.postgresql.org/patch/7215/) (cf, Withdrawn, 2026-08-29) - judged not relevant
- P17: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P18: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P19: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P20: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant
- P21: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P22: [repair or remove check_guc script](https://commitfest.postgresql.org/patch/3441/) (cf, Withdrawn, 2021-12-29) - judged not relevant
- P23: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P24: [COPY FREEZE and PD_ALL_VISIBLE](https://commitfest.postgresql.org/patch/401/) (cf, Returned with feedback, 2015-12-24) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 92 s; cost reported by the CLI: $0.87
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:53:32.964Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
