# Design pre-review: Fall back to serializing when parallel apply runs out of DSM segments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7548](https://commitfest.postgresql.org/patch/7548/) (Needs review); authors: Baji Shaik (bajishaik)
- Thread: [\[PATCH\] Fall back to serializing when parallel apply runs out of DSM segments](https://www.postgresql.org/message-id/flat/CA%2Bfm-RMyb6EoMuk%2Ba4_hZou9cdTHfB6mj1UV-7yb_YuTWLug1g%40mail.gmail.com), started 2026-10-23
- Messages read: 5 of 5
- Patch prose read: v2-0002-Make-DSA-honor-DSA_ALLOC_NO_OOM-when-out-of-DSM-s.patch, v2-0001-Fall-back-to-serializing-when-parallel-apply-runs.patch, 0001-Fall-back-to-serializing-when-parallel-apply-runs-ou.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. Both patches pass a flag that InitializeParallelDSM() already passes, so existing NULL checks and the documented streaming = parallel fallback become reachable. Message #1 gives a reproduction, which P4 and P5 lacked. Confidence is medium because the code was not read and the callers of make_new_segment() were not checked.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. Whether the leader's fallback and every caller of make_new_segment() behave correctly on the newly reachable NULL path could not be judged.

## The proposal, restated

**Problem.** With streaming = parallel, the leader apply worker errors out when no DSM segment is free, because dsm_create() raises "too many dynamic shared memory segments" instead of returning NULL. The worker restarts and fails at the same STREAM START, so the subscription makes no progress until segments are freed. The same flaw makes DSA_ALLOC_NO_OOM raise an error instead of returning InvalidDsaPointer.

**Proposed design.** 0001 passes DSM_CREATE_NULL_IF_MAXSEGMENTS in pa_setup_dsm(), as InitializeParallelDSM() does. The existing fallback then runs, and the leader serializes the streamed transaction to a file and applies it itself. 0002 passes the same flag in make_new_segment() in dsa.c, so the existing NULL check there is reached.

**User-visible surface.**
- Behavior change: with streaming = parallel, running out of DSM segments now spills the transaction to temporary files instead of failing the apply worker.
- C API behavior change: dsa_allocate_extended() with DSA_ALLOC_NO_OOM returns InvalidDsaPointer when DSM segments run out, instead of raising an ERROR.

**What the author claims.**
- The patch makes the server match the documented streaming = parallel fallback to temporary files.
- In a test that held all DSM slots, master restarted the apply worker 13 times in a minute with nothing applied; the patch applied all rows.
- Without 0002, callers such as pgstat_get_entry_ref() and pg_stash_advice skip their cleanup when DSM segments run out.
- The flaw dates from 216a784829c; 0001 applies to 16 and later, and 0002's code is the same on all supported branches.
- The author first proposed backpatching 0001 to 16 and 0002 to 14.

**Thread state.** Zhijie Hou and Bharath Rupireddy both approve the patches; v2 drops a comment change at Hou's request. Bharath argues for HEAD only because the case is rare and unreported, the author accepts that, and the backpatch decision is left to a committer.

## 2. Prior attempts

### Improve logicalrep_worker_launch() logic (related)

- **Link:** [https://commitfest.postgresql.org/patch/5734/](https://commitfest.postgresql.org/patch/5734/)
- **Thread:** [https://www.postgresql.org/message-id/flat/d9e208a0-1bdf-42e3-b6c6-49bca1904060@oss.nttdata.com](https://www.postgresql.org/message-id/flat/d9e208a0-1bdf-42e3-b6c6-49bca1904060@oss.nttdata.com)
- **Date:** decided 2026-01-08; started 2025-04-25
- **Outcome:** Withdrawn (cf)
- **Who objected:** Amit Kapila (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Amit Kapila argued that the parallel apply case is not a clear bug, because the apply worker applies changes itself when no parallel apply worker is available. He asked for a test case that reproduces the problem rather than a change based on reading the code. The thread ends with that request unanswered, and the entry was withdrawn about eight months later. No message states the reason for the withdrawal.
- **Reason (quote):** "No, the idea was that if we are changing the code, it is better to test it via a testcase instead of changing based on assumptions." ([message](https://www.postgresql.org/message-id/CAA4eK1J17Le_Xx4BmGXa-KubmuBaw7xRhvok7WqzsY_5heyK%2BA%40mail.gmail.com))
- **What is different this time:** P4 changed worker-slot cleanup based on reading the code. This proposal makes an existing fallback reachable and reports a reproduction in message #1.

### logicalrep_worker_launch - counting/checking the type limits (related)

- **Link:** [https://commitfest.postgresql.org/patch/4499/](https://commitfest.postgresql.org/patch/4499/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAHut+Pu-7DiCwMdbBf5vzUL1t8H=_NpyFR4RvpZbpMGXV=+QLQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAHut+Pu-7DiCwMdbBf5vzUL1t8H=_NpyFR4RvpZbpMGXV=+QLQ@mail.gmail.com)
- **Date:** decided 2024-04-23; started 2023-08-11
- **Outcome:** Withdrawn (cf)
- **Who objected:** Amit Kapila (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the entry after a long period without activity. Amit Kapila asked whether the cleanup condition could be reproduced and whether a crashed leader worker would restart the server anyway, and the author did not answer. The entry moved to Waiting on Author, and the author withdrew it when the commitfest manager asked him to answer or withdraw.
- **Reason (quote):** "Did you try to reproduce this condition, if not, can you please try it once? I wonder if the leader worker crashed, won't it lead to a restart of the server?" ([message](https://www.postgresql.org/message-id/CAA4eK1LtFyiMV6e9%2BRr66pe5e-MX7Pk6N3iHd4JgcBW1X4kS6Q%40mail.gmail.com))
- **What is different this time:** P5 changed per-type worker limit checks without a reproduction. This proposal reproduced the failure, which is a leader apply worker restart loop.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0001 changes what the leader apply worker does at STREAM START when no DSM segment is free. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0002 makes dsa_allocate_extended() with DSA_ALLOC_NO_OOM return InvalidDsaPointer where it raised an ERROR, which extension callers see. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The commit messages say backpatch to 16 and 14, while the thread now leans to HEAD only and leaves the choice to a committer. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does every path that reaches make_new_segment() handle a NULL segment correctly now that 0002 makes that return reachable?
2. Can the out-of-segments path get an in-tree regression test, since the reproduction in message #1 used an external test extension?
3. Do other dsm_create() callers have the same unreachable NULL check, and should this series fix them too?
4. Will the commit messages drop the Backpatch-through lines if the fix goes to HEAD only?
5. What is pg_stash_advice, and where does the code that 0002's commit message names live?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. 0001 needs a committer who knows the logical replication apply code; 0002 needs one who knows DSA and DSM. The brief names two reviewers who approve but does not show a committer in the thread.
- **Liability:** The owner signs for the leader's serialize fallback on a path that was never reachable before, and for every DSA_ALLOC_NO_OOM caller's NULL handling. A backpatch would extend that to minor releases of 14 and later.
- **Author follow-up:** The author sent v2 within a day of the first review, added 0002 on request, and accepted the HEAD-only suggestion.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Each patch is under 2 KiB, touches one file and adds no new user-visible surface. Only subscribers that run out of DSM segments with streaming = parallel benefit, and two reviewers already approve.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found for this idea. P4 and P5 died for lack of a reproduction, and message #1 supplies one. Its repro used an external extension, and the patches add no in-tree regression test; that is a level 4 matter, outside this angle.
- standards-compat: 0 objection(s). Both patches make the server follow the contract already in the docs: the streaming = parallel fallback to temporary files, and InvalidDsaPointer under DSA_ALLOC_NO_OOM. No syntax, GUC, catalog, pg_dump or psql surface changes, and only callers that already handle the documented failure value see new behavior. Back-patching 0002 to minor releases would let extension callers reach their NULL-handling paths for the first time, but the thread now proposes HEAD only, and whether those paths are correct is a code question.
- replication-ondisk: 0 objection(s). Neither patch changes on-disk format, WAL, catalogs, the wire protocol or struct layouts. 0001 sends the leader into the existing serialize-to-file fallback, which already runs when no parallel apply worker is free; 0002 lets DSA_ALLOC_NO_OOM callers get the InvalidDsaPointer result their contract already promises. Not checkable without code: whether every make_new_segment() caller handles NULL.
- security: 0 objection(s). Both patches add DSM_CREATE_NULL_IF_MAXSEGMENTS so an existing NULL check becomes reachable; nothing changes who may do what or see what. Running out of DSM segments now spills to the same temporary files that streaming = on already uses, so no new disk-use or denial-of-service path for unprivileged users appears.
- maintenance-fit: 0 objection(s). Both patches add a flag that parallel query already passes to dsm_create(), so existing NULL checks and the documented fallback to temporary files become reachable. Nothing new is added on a common path, and there is no new API, GUC or dependency. Without the code I cannot confirm that every make_new_segment() caller handles a NULL segment.
- already-exists: 0 objection(s). This is a bug fix that makes existing fallback code reachable, using the flag DSM_CREATE_NULL_IF_MAXSEGMENTS that InitializeParallelDSM() already passes. It adds no mechanism, and no existing feature or extension meets the need in its place, so there is nothing to raise from the already-exists angle.
- demand: 0 objection(s). Demand is not the right test for this proposal. It is a small bug fix that makes streaming = parallel match its documented fallback to temporary files, and it makes the existing NULL checks in pa_setup_dsm() and make_new_segment() reachable. The thread's own concern about demand (message #4: rare, not reported from the field) was settled by applying the fix to HEAD only, which the author accepted in message #5. Unlike [P4] and [P5], the author gave a reproduction: on master the apply worker restarted 13 times in a minute with nothing applied, and with the patch all rows were applied. The cost falls only on the out-of-segments path, so users who never reach it pay nothing. One question is for other panel members: whether every caller of make_new_segment() handles a NULL segment once 0002 makes that path reachable.

### Precedents retrieved

- P1: [DSA Hash Tables (DHT): hash tables backed by DSM segments](https://commitfest.postgresql.org/patch/862/) (cf, Returned with feedback, 2016-12-21) - judged not relevant; reason summarised from its thread
- P2: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P3: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P5: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - used in the review; reason summarised from its thread
- P6: [Implement NULL-related checks in object address functions to prevent cache lookup errors](https://commitfest.postgresql.org/patch/1204/) (cf, Returned with feedback, 2018-11-29) - judged not relevant; reason summarised from its thread
- P7: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P8: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P9: [Improve monitoring of shared memory allocations](https://commitfest.postgresql.org/patch/5620/) (cf, Rejected, 2026-04-07) - judged not relevant; reason summarised from its thread
- P10: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P11: [Fallback table AM for relkinds without storage](https://commitfest.postgresql.org/patch/2986/) (cf, Returned with feedback, 2021-03-14) - judged not relevant
- P12: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P13: [Use AF_UNIX for tests on Windows (ie drop fallback TCP code)](https://commitfest.postgresql.org/patch/4033/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P14: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P15: [Fix bug of COPY (on_error set_null)](https://commitfest.postgresql.org/patch/6774/) (cf, Withdrawn, 2026-05-19) - judged not relevant
- P16: [Allow pg_upgrade to copy segments of the same relfilenode in parallel](https://commitfest.postgresql.org/patch/3525/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P17: [Revert recent change to RequestNamedLWLockTranche().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d814d7fc3d5257ae258b502229fc7ca97c97270a) (revert, Reverted, 2025-09-04) - judged not relevant
- P18: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P19: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P20: [Incorrect handling of OOM in WAL replay leading to data loss](https://commitfest.postgresql.org/patch/4590/) (cf, Withdrawn, 2023-10-24) - judged not relevant
- P21: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P22: [Check for tuplestorestate nullness before dereferencing](https://commitfest.postgresql.org/patch/5705/) (cf, Withdrawn, 2025-12-08) - judged not relevant
- P23: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P24: [tracking last allocated XLOG segment in control data](https://commitfest.postgresql.org/patch/444/) (cf, Rejected, 2016-01-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 75 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 49 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:08:22.681Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
