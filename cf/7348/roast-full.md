# Design pre-review: Table sync race with REFRESH PUBLICATION

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7348](https://commitfest.postgresql.org/patch/7348/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [\[PATCH\] Table sync race with REFRESH PUBLICATION](https://www.postgresql.org/message-id/flat/CAJTYsWWJfq%3DM5U9h7sJFcQ%2BLPwq76CzLTzQbkRQPTJJ47SGxWw%40mail.gmail.com), started 2026-09-25
- Messages read: 6 of 6
- Patch prose read: v3-0001-Recheck-table-sync-state-after-refresh.patch, v3-0002-Test-table-sync-after-a-concurrent-refresh.patch, 0002-Test-table-sync-after-a-concurrent-refresh.patch, 0001-Recheck-table-sync-state-after-refresh.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproduction, and the fix re-reads one pg_subscription_rel state under a lock the apply worker already takes, following the sequence-side fix in dca73f7dd03. Confidence is medium because no code was read and only one reviewer has confirmed that the state check is enough without the LSN check.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: No code was read. Two points depend on it: whether the state check alone covers the remove and re-add case, and whether the refresh drops the origin and slot that the apply worker now skips.

## The proposal, restated

**Problem.** An apply worker caches a table's SYNCDONE state, then waits for the subscription lock while ALTER SUBSCRIPTION ... REFRESH PUBLICATION removes the table. The worker then tries to mark the missing pg_subscription_rel row READY and errors out, and with disable_on_error the whole subscription is disabled. If two refreshes remove and re-add the table, the old state could skip the new initial copy.

**Proposed design.** After taking the subscription lock in ProcessSyncingTablesForApply(), the apply worker reads the table's sync state again. It skips origin cleanup and the READY update unless the state is still SYNCDONE, and logs a DEBUG1 message when it skips. v1 also rechecked the sync LSN, but v2 dropped that check after review.

**User-visible surface.**
- New DEBUG1 log message when the apply worker skips the READY transition for a removed or re-added table.
- Test patch adds an injection point in tablesync.c before the apply worker locks the subscription.

**What the author claims.**
- The race disables the whole subscription when disable_on_error = true, including unrelated tables.
- An existence-only check is not enough, because a removed and re-added table would skip its new initial copy.
- An LLM-assisted review reproduced the missing-row error, and the author tested the remove/re-add case by hand.
- There is no field report of the issue that the author knows of.
- The table-side issue predates the sequence-side fix in dca73f7dd03.

**Thread state.** Peter Smith reviewed v1, the author applied his comments in v2, and Peter Smith called 0001 good but asked for more reviewers. The author posted v3 with simpler tests and said the test patch is not meant to be committed.

## 2. Prior attempts

### Fix alter subscription concurrency errors (related)

- **Link:** [https://commitfest.postgresql.org/patch/3842/](https://commitfest.postgresql.org/patch/3842/)
- **Thread:** [https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com](https://www.postgresql.org/message-id/flat/HE1PR8303MB0075BF78AF1BE904050DA16BF7729@HE1PR8303MB0075.EURPRD83.prod.outlook.com)
- **Date:** decided 2023-03-28; started 2022-08-25
- **Outcome:** Withdrawn (cf)
- **Who objected:** Amit Kapila, Alvaro Herrera, Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected that adding lock-and-retry lookup code specific to subscriptions does not scale, since the same problem affects other object types. Alvaro Herrera suggested using get_object_address() instead, and Tom Lane added that this might first require improving its error reporting. The author did not post an updated version for several commitfests. He then said his own test-flakiness problem had been worked around and that he was not motivated to address the issue for all DDL commands, and the entry was withdrawn.
- **Reason (quote):** "I think the patch as-submitted is pretty uninteresting, mainly because the design of adding bespoke lock code for subscription objects doesn't scale." ([message](https://www.postgresql.org/message-id/956468.1679335244%40sss.pgh.pa.us))
- **What is different this time:** P3 added new lock-and-retry lookup code for subscription DDL, modeled on dbcommands.c. This proposal adds no lock code: it re-reads one pg_subscription_rel state under the existing lock in the apply worker, following the sequence-side fix in dca73f7dd03.


## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The fix changes how the logical replication apply worker moves a table from SYNCDONE to READY in tablesync.c. |
| wire protocol | no |  |
| catalog changes | **yes** | The apply worker re-reads the pg_subscription_rel state under the subscription lock; the catalog schema does not change. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable branches, but the thread does not name the affected branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | v3-0002 adds an injection point in tablesync.c that would ship if the test patch were committed. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches have the race, and which branches does the fix target for back-patching?
2. When the apply worker skips origin cleanup, does ALTER SUBSCRIPTION ... REFRESH PUBLICATION still drop the tablesync origin and slot for the removed table?
3. Can a re-added table reach SYNCDONE again while the apply worker still waits for the subscription lock, which would make the state check alone insufficient?
4. Should the remove and re-add case have an automated test, given that it protects against a skipped initial copy?
5. If the test patch is not committed, what keeps this recheck from being removed by a later change to ProcessSyncingTablesForApply()?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the logical replication tablesync and apply worker code. The visible thread shows one reviewer, Peter Smith, who asked for more reviewers.
- **Liability:** The owner signs for the SYNCDONE to READY step on every branch that gets the fix, including that a skipped READY update never leaves a table without its initial copy or an orphaned origin.
- **Author follow-up:** The author posted v2 the same day as the review and then v3 with simpler tests, and offered to send more tests on request.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The race needs a REFRESH PUBLICATION at a narrow moment and has no field report, but with disable_on_error it disables the whole subscription. The fix is about 2.4 KB in one file and follows an existing pattern, so a second review by someone who knows tablesync.c is cheap.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent's recorded reason applies. P3 died over new subscription-specific lock code, and this proposal adds none. P2 reverted a fix that raised an error in ALTER SUBSCRIPTION ... REFRESH SEQUENCES, while this proposal only skips the READY update in the worker and leaves the user command alone. The thread itself cites dca73f7dd03 as the model (messages #1 and #2), not as an earlier failed attempt.
- standards-compat: 0 objection(s). From the standards and compatibility angle, this bug fix has nothing to object to. It adds no syntax, GUC, catalog object or psql change, and dump/restore is not involved; the only user-visible change is a DEBUG1 message, and stopping the subscription from being disabled is the intended result. The thread does not yet name the target back branches, and the v3 test would add an injection point to tablesync.c, but the author says the test is not meant to be committed.
- replication-ondisk: 0 objection(s). The fix adds a catalog re-read under an existing lock and changes no on-disk format, WAL record, catalog schema, protocol message or exported API, so back-patching is not blocked by ABI. Not checkable without code: that ALTER SUBSCRIPTION ... REFRESH PUBLICATION still drops the tablesync origin and slot when the apply worker skips origin cleanup.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Only the subscription owner or a superuser can run ALTER SUBSCRIPTION ... REFRESH PUBLICATION, and the fix only makes the apply worker skip a catalog update; it adds no new grant, role or default. The DEBUG1 message goes to the server log and names only the relation OID and subscription. The test injection point exists only in builds with injection points enabled.
- maintenance-fit: 0 objection(s). From the maintenance angle, this is a small recheck of the sync state under a lock the code already takes. It follows the pattern of the sequence-side fix in dca73f7dd03 and adds a catalog read only on the rare SYNCDONE to READY step, so it adds no hot-path cost and no new API, GUC or catalog. [P3] does not apply because the patch adds no new lock code. [P2] concerns blocking a user command and does not apply either. Two committability points remain: no committer has engaged yet, since only Peter Smith reviewed (message #4), and v3-0002 adds an injection point in tablesync.c that would ship if the test patch were committed, although message #5 says it is not meant to be.
- already-exists: 0 objection(s). This is a bug fix inside the apply worker, and no existing feature, setting or extension can stop the worker from updating a pg_subscription_rel row that is gone. The fix follows the sequence-side recheck in dca73f7dd03 and adds no second mechanism, so item 12 is met, and [P3] (bespoke lock code) does not apply because the patch reuses the existing subscription lock.
- demand: 0 objection(s). This is a small bug fix in tablesync.c, not a feature, so the demand test does not apply in the usual way. The material shows a real failure: a concurrent REFRESH PUBLICATION makes the apply worker error out, disable_on_error then disables the whole subscription, and a re-added table can skip its initial copy. Message #4 confirms the reproduction. The fix adds one catalog read after a lock the worker already takes, so non-users pay almost nothing. The missing field report (messages #1 and #5) weakens the case for the injection-point test, not for the fix.

### Precedents retrieved

- P1: [Apply worker fails if a relation is missing on subscriber even if refresh publication has not been refreshed yet](https://commitfest.postgresql.org/patch/4077/) (cf, Withdrawn, 2022-12-27) - judged not relevant; reason summarised from its thread
- P2: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - used in the review
- P3: [Fix alter subscription concurrency errors](https://commitfest.postgresql.org/patch/3842/) (cf, Withdrawn, 2023-03-28) - used in the review; reason summarised from its thread
- P4: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P5: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P6: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P7: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P8: [Subscription code improvements](https://commitfest.postgresql.org/patch/1248/) (cf, Returned with feedback, 2018-04-07) - judged not relevant; reason summarised from its thread
- P9: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P10: [Generic type subscription](https://commitfest.postgresql.org/patch/793/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P11: [PG 18 beta1 release notes misses mention of pg_noreturn](https://commitfest.postgresql.org/patch/5895/) (cf, Rejected, 2025-09-01) - judged not relevant
- P12: [Logical Replication - revisit `is_table_publication` function implementation](https://commitfest.postgresql.org/patch/7042/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P13: [Revert "Validate subscription conninfo on owner change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ac230cd3ce93c4752489564cfe6b37b584a465ac) (revert, Reverted, 2026-08-05) - judged not relevant
- P14: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P15: [Revert "vacuumdb: Add option for analyzing only relations missing stats."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19e57f4f78e4354d9a21c284868373d28bb0d368) (revert, Reverted, 2025-03-07) - judged not relevant
- P16: [Fix TOCTOU race in ReplicationSlotsComputeRequiredLSN()](https://commitfest.postgresql.org/patch/6682/) (cf, Withdrawn, 2026-04-17) - judged not relevant
- P17: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P18: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant
- P19: [Bug in ALTER SUBSCRIPTION ... SERVER / ... CONNECTION with broken old server](https://commitfest.postgresql.org/patch/6707/) (cf, Withdrawn, 2026-06-22) - judged not relevant
- P20: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant
- P21: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P22: [Revert "Avoid creating archive status ".ready" files too early"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96b665083eb72570e226cf2d25c960b3acc62040) (revert, Reverted, 2021-09-04) - judged not relevant
- P23: [Revert part of recent include patch not ready for application.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=51e888219745cf9efa41034b5411eee7e9a8f4f1) (revert, Reverted, 2006-07-14) - judged not relevant
- P24: [Revert ill-fated patch to release exclusive lock early after vacuum truncates a table. Introduces race condition, as shown by buildfarm failures.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=eb5f4d6c5cece3e3fa80a506153e5202b093b7ca) (revert, Reverted, 2007-09-20) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 69 s; cost reported by the CLI: $0.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:43:03.206Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
