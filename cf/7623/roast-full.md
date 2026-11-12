# Design pre-review: Injection point: fix race condition between attach and cleanup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7623](https://commitfest.postgresql.org/patch/7623/) (Needs review); authors: Hayato Kuroda (ha-kun)
- Thread: [Injection point: fix race condition between attach and cleanup](https://www.postgresql.org/message-id/flat/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com), started 2026-11-11
- Messages read: 3 of 3
- Patch prose read: v2-0001-injection-points-reproducer-for-the-race-conditio.patch, v2-0002-injection-points-Add-handles-for-detaching-callba.patch, nocfbot-0001-Stabilize-010_index_concurrently_upsert.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection stops the proposal at levels 1 or 2. The race is real and the 0001 spec reproduces it; the test-only fix leaves it in the module. demand-1 and standards-compat-1 are both fixable: the thread names one failing test as the consumer of a new core API, and names no branch for either fix. A committer who knows injection points must choose between the two fixes, which the author asks for in message #1.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot check whether the shared-memory slot layout changes, how the generation counter handles slot reuse, or whether the handle check and the detach happen under one lock.

## The proposal, restated

**Problem.** The injection_points test module removes points attached after injection_points_set_local() in a before_shmem_exit callback, and finds them only by name. If another backend detaches the point and attaches a new callback under the same name before the first backend exits, that cleanup detaches the new callback. The author reports that this makes the test 010_index_concurrently_upsert fail at times.

**Proposed design.** The patch adds InjectionPointHandle, which records the shared-memory slot and its generation for one attach call. InjectionPointDetachByHandle() detaches the callback only while the handle still matches, so a stale cleanup cannot remove a later callback. The test module stores handles for local points and detaches by handle at exit, and the name-based interfaces stay unchanged.

**User-visible surface.**
- C API: new type InjectionPointHandle.
- C API: new function InjectionPointAttachWithHandle(name, library, function, private_data, private_data_size, handle).
- C API: new function InjectionPointDetachByHandle(handle), which returns false if that callback is already detached.
- Docs: xfunc.sgml documents the two new functions.
- Tests: new isolation spec local_cleanup in the injection_points module, which reproduces the race.

**What the author claims.**
- The race can detach a callback that another backend attached later under the same name.
- The 0001 isolation spec reproduces the race.
- The fix leaves the existing name-based attach and detach interfaces unchanged.
- A smaller fix only for 010_index_concurrently_upsert waits until cleanup is done; it was posted as a nocfbot patch.

**Thread state.** Only the author has posted, in three messages: the first proposal and two v2 reposts to fix cfbot and a bad attachment. No reviewer has replied, and the author asks for a choice between the core API fix and the test-only fix.

## 2. Prior attempts

### Disable installcheck for tap and regress tests for injection_points module (related)

- **Link:** [https://commitfest.postgresql.org/patch/5410/](https://commitfest.postgresql.org/patch/5410/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANtu0oiTgFW47QgpTwrMOVm3Bq4N0Y5bjvTy5sP0gYWLQuVgjw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANtu0oiTgFW47QgpTwrMOVm3Bq4N0Y5bjvTy5sP0gYWLQuVgjw@mail.gmail.com)
- **Date:** decided 2025-02-21; started 2024-11-25
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the entry after noting that runningcheck was already disabled for the injection_points isolation tests in the tree, which covered most of the patch. Earlier, Michael Paquier objected that the crash may point to a real bug and that disabling the tests hides it. The author replied that the cause is injection_points shared memory setup during abort, but no fix for that cause was agreed in the thread.
- **Reason (quote):** "Disabling the test hides the problem, it does not fix it." ([message](https://www.postgresql.org/message-id/Z0lL3GKCyijL8HcI%40paquier.xyz))
- **What is different this time:** P4 turned off installcheck for the injection_points tests to avoid a crash in shared memory setup during abort. This proposal fixes a different race, in the cleanup of local points, and addresses its cause in the module.


## 3. Objections a reviewer is likely to raise

### 1. New core API has one consumer while a test-only fix exists

fixable · level 1b, rubric item 3 · demand, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds InjectionPointAttachWithHandle() and InjectionPointDetachByHandle() to core injection_point.c, and only the injection_points test module ever calls them. Later work on injection_point.c must keep the slot and generation rules behind the handle intact, though only one cleanup path depends on them.

The material shows one concrete failure: 010_index_concurrently_upsert fails at times (message #1). Message #1 also offers a test-only fix that waits for cleanup, and the author asks reviewers to choose between the two. No other test, extension or user is named as needing to detach by handle, and the docs present it as a general API. A committer would likely ask whether more than one test is exposed before taking the larger fix.

Thread: [#1 ("Hayato Kuroda (Fujitsu)", 2026-11-11)](https://www.postgresql.org/message-id/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com)

**Question:** Which tests or extensions besides 010_index_concurrently_upsert use injection_points_set_local() in a way that can hit this race, and how would the test-only fix leave them exposed?

Counter-check: held. The 0001 spec shows the race is in the module's cleanup, so the test-only fix leaves it in place. But the material names only one failing test, and the author asks reviewers to choose between the two fixes.

### 2. Core API addition does not say which branches get the fix

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API · `standards-compat-1`

010_index_concurrently_upsert keeps failing at times on the buildfarm in a stable branch. A committer must then either back-patch a new exported type and two functions in injection_point.h, or apply the test-only fix there. Master and the back branches then clean up local points in different ways.

The fix adds a new public C type and two functions to backend code to fix a race in a test module. The thread does not say which branches carry the failing test or the local-point cleanup, so the cost of the core change in stable branches is unknown. The author offers a test-only alternative but does not say whether it is meant for the back branches, which would mean two fixes for one race. A committer would likely ask for this plan before taking on new API in core.

Thread: [#1 ("Hayato Kuroda (Fujitsu)", 2026-11-11)](https://www.postgresql.org/message-id/OS7PR01MB183175FA935E5DC8734FF2CD6F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com), [#2 ("Hayato Kuroda (Fujitsu)", 2026-11-11)](https://www.postgresql.org/message-id/OS7PR01MB18317388DDFE410B1B93562F3F5712%40OS7PR01MB18317.jpnprd01.prod.outlook.com)

**Question:** Which branches have this race, and which fix, the handle API or the nocfbot test change, goes into each of them?

Counter-check: held. The thread offers two fixes and names no branch for either. Injection points exist only in builds configured with them, and new functions do not break ABI, so the back-branch cost is likely small, but the plan is still missing.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds InjectionPointHandle and two exported functions to injection_point.h and documents them in xfunc.sgml. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The thread does not say which branches have the race, and a back-patch would add a new type and two functions. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Later changes to injection_point.c must keep the slot and generation rules that the handle depends on. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which tests or extensions besides 010_index_concurrently_upsert use injection_points_set_local() in a way that can hit this race?
2. Which branches have this race, and which fix, the handle API or the nocfbot test change, goes into each of them?
3. Is the handle API meant for extensions outside the injection_points test module, or only for that module?
4. How does the generation counter behave when a slot is reused many times, including on wraparound?
5. Does the same race exist for points that are not local, when two backends detach and attach the same name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the injection point code in injection_point.c and the injection_points test module. The author copied two people from the original thread; neither has replied.
- **Liability:** The owner signs for a new documented C type and two functions in core, and for the slot and generation rules behind them. The code exists only in builds with injection points enabled, so the exposure is limited to test builds.
- **Author follow-up:** The author posted two corrected versions within a day to fix cfbot and a bad attachment, and asks for feedback on which fix to pursue.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch fixes a reproducible race that makes a test fail at times, and it benefits developers who write tests with local injection points. It is about 13 KiB, touches one core file, and keeps the name-based interfaces unchanged.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent rejects this idea. The reason recorded for [P4] applies only to the nocfbot test-only fix in message #1, which works around the race in one test and leaves it in the module; the main proposal (0002) fixes the cause.
- standards-compat: 1 objection(s). From the standards and compatibility angle the proposal is small. It keeps the name-based interfaces and adds no SQL syntax, GUC, catalog entry or dump state. The only open point is which branches get which fix.
- replication-ondisk: 0 objection(s). From this angle the proposal has nothing to object to. Injection point state lives only in shared memory, is rebuilt at every server start, and is never written to WAL, relation files or catalogs, so pg_upgrade, standbys and subscribers do not see it. The patch adds a type and two functions to injection_point.h and leaves the existing name-based attach and detach functions as they are, so no current extension call breaks. Without the code, two points cannot be checked: whether the shared-memory slot layout changes, and whether a back-patch would change anything beyond adding these functions.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Injection points are a test facility that only works in builds with injection points enabled, and the new handle API changes no privilege, adds no SQL-callable surface and exposes no data. A stale handle can only fail to detach, so it gives no new way to affect other sessions.
- maintenance-fit: 0 objection(s). From the maintenance-and-fit angle there is no objection. The new API exists only in builds with injection points enabled, so it costs nothing on any hot path. It fixes a real race in name-based detach, it is optional, and the existing interfaces stay unchanged. The only open point is committability: no committer has replied yet, and the author asks whether a core API fix or a test-only fix is preferred. A committer would likely ask whether the handle is meant for out-of-core extensions, since that decides how stable it has to stay. That needs an answer, but it is not a failure scenario. [P4] concerns installcheck crashes in the same module and does not apply here.
- already-exists: 0 objection(s). No existing core feature, contrib module or external tool detaches one specific injection point callback safely; InjectionPointDetach() only matches by name, which is the race itself. The author's test-only fix stabilizes 010_index_concurrently_upsert but not the module, so whether the core handle API is worth adding is a desirability and design question for other angles.
- demand: 1 objection(s). The flaky test is real demand for some fix. The open point is whether that demand justifies a core API addition.

### Precedents retrieved

- P1: [Persist injection points across server restarts](https://commitfest.postgresql.org/patch/5731/) (cf, Returned with feedback, 2025-07-02) - judged not relevant; reason summarised from its thread
- P2: [Backpatching injection point core facilities to REL_17_STABLE](https://commitfest.postgresql.org/patch/5865/) (cf, Withdrawn, 2025-08-17) - judged not relevant; reason summarised from its thread
- P3: [Some refactoring for fixed-numbered stats template in injection_points](https://commitfest.postgresql.org/patch/5187/) (cf, Withdrawn, 2024-10-02) - judged not relevant; reason summarised from its thread
- P4: [Disable installcheck for tap and regress tests for injection_points module](https://commitfest.postgresql.org/patch/5410/) (cf, Withdrawn, 2025-02-21) - used in the review; reason summarised from its thread
- P5: [Revert "Add recovery TAP test for race condition with slot invalidations"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=65db0cfb4c036b14520a22dba5a858185b713643) (revert, Reverted, 2024-03-07) - judged not relevant
- P6: [Revert "Fix partition pruning setup during DETACH CONCURRENTLY"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b0ea16528cda1f3a993bbfeb2b93eed9762af4e6) (revert, Reverted, 2024-06-24) - judged not relevant
- P7: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P8: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P9: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant; reason summarised from its thread
- P10: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P11: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P12: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P13: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant
- P14: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P15: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/2496/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P16: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant
- P17: [Cleanup gcc trick with varattrib_1b_e in VARATT_EXTERNAL_GET_POINTER()](https://commitfest.postgresql.org/patch/5807/) (cf, Withdrawn, 2025-07-14) - judged not relevant
- P18: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P19: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant
- P20: [Generate Doxygen documentation](https://commitfest.postgresql.org/patch/4641/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P21: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P22: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant
- P23: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant
- P24: [avoid bloat from CREATE INDEX CONCURRENTLY](https://commitfest.postgresql.org/patch/1047/) (cf, Returned with feedback, 2017-03-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 124 s; cost reported by the CLI: $1.11
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 59 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:09:32.796Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
