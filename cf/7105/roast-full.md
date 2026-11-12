# Design pre-review: Add tests for concurrent DML retry paths in logical replication apply

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7105](https://commitfest.postgresql.org/patch/7105/) (Needs review); authors: Bharath Rupireddy (bharathrupireddy)
- Thread: [Add tests for concurrent DML retry paths in logical replication apply](https://www.postgresql.org/message-id/flat/CALj2ACV6ESpggn2Az%3DOdZBzVx7jiBG9O_EdKdgbk2chAJusC2w%40mail.gmail.com), started 2026-04-27
- Messages read: 5 of 5
- Patch prose read: v3-0001-Add-tests-for-concurrent-DML-retry-paths-in-logic.patch, v1-0001-Add-tests-for-concurrent-DML-retry-paths-in-logic.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No objection to the tests survived: the coverage report shows the retry paths are untested, and injection points cost production builds nothing. The two surviving objections, maintenance-fit-1 and maintenance-fit-2, concern only the bundled slot cleanup in execReplication.c. It has no measured gain, and its safety argument in message #5 rests on heap buffer pins and is unconfirmed by any reviewer. Both are fixable, most directly by the split the author already offers.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check that neither retry path reads the slot after the buffer pin could be released, or that the test is stable. Messages #4 and #5 were seen only through the objections.

## The proposal, restated

**Problem.** The retry paths in RelationFindReplTupleByIndex and RelationFindReplTupleSeq have no test coverage, according to the coverage report. These paths run when the apply worker finds that the target tuple is being changed concurrently on the subscriber. The audience is developers of logical replication.

**Proposed design.** A new TAP test covers both retry paths. It holds an in-progress transaction on the subscriber for the XactLockTableWait path, and it uses an injection point that pauses the worker between finding and locking the tuple for TM_Updated and TM_Deleted. The patch also makes RelationFindReplTupleSeq scan directly into outslot, and both functions now materialize the slot only once, before returning it.

**User-visible surface.**
- New injection point(s) in execReplication.c before table_tuple_lock; the names are not given in the material.
- New TAP test src/test/subscription/t/039_concurrent_dml_retry.pl.

**What the author claims.**
- The retry paths in both lookup functions had no test coverage.
- Earlier materialization is unnecessary because the scan keeps the buffer pinned and both retry paths restart the lookup.
- table_tuple_lock needs only the TID, so the slot need not be materialized before the lock.
- Dropping the extra slot and ExecCopySlot in RelationFindReplTupleSeq avoids a redundant tuple copy.
- The author says these are not major performance issues in practice; there are no numbers.

**Thread state.** Hayato Kuroda reviewed v1 and v2 and questioned the materialization change; the author answered with v3 on 2026-06-27. Nobody has replied since, and the CommitFest entry still needs review after one move.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Unmeasured slot cleanup bundled with tests in rarely run apply code

fixable · level 1b, rubric item 4 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A committer pushes v3 for its tests. The removed materialization later turns out to matter in some case, and an apply worker on a subscriber writes a row built from stale or wrong values. Users see this as silent divergence between publisher and subscriber, long after the commit.

The tests add coverage at no runtime cost, but the execReplication.c cleanup changes behaviour in the conflict path that the tests are meant to protect. Message #1 says the gain is not a "major performance issue in practice", and there are no numbers. In message #4 the only reviewer questioned the materialization change, and nobody has confirmed the reply in message #5. The author offers to split the patch in message #5, and nobody has answered.

Thread: [#1 (Bharath Rupireddy, 2026-04-27)](https://www.postgresql.org/message-id/CALj2ACV6ESpggn2Az%3DOdZBzVx7jiBG9O_EdKdgbk2chAJusC2w%40mail.gmail.com), [#4 ("Hayato Kuroda (Fujitsu)", 2026-06-25)](https://www.postgresql.org/message-id/OS9PR01MB121491200B47154926F826084F5EC2%40OS9PR01MB12149.jpnprd01.prod.outlook.com), [#5 (Bharath Rupireddy, 2026-06-27)](https://www.postgresql.org/message-id/CALj2ACWHZuGSU2m-DkStZYCvv0WxWAzVSnt9w1BZbRzvx0D2xg%40mail.gmail.com)

**Question:** Can the tests be committed first, with the cleanup as a separate patch backed by numbers or a concrete reason beyond consistency?

Counter-check: held. Message #1 states the gain is small, no numbers exist, and no reviewer confirmed the safety argument in message #5. The author's offer to split the patch supports the request and remains unanswered.

### 2. Safety argument for dropping materialization assumes heap buffer slots

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

An extension table AM whose slots do not hold a buffer pin is the target of logical replication. After v3, RelationFindReplTupleSeq scans straight into outslot and skips the earlier materialization. The AM author finds wrong or invalid tuple data only when replicated updates misbehave in the field.

Message #5 justifies the change by the pin that TTSOpsBufferHeapTuple keeps and by clock-sweep behaviour, which are heap and buffer-manager facts. execReplication.c works through the table AM API, so the invariant has to hold for any AM slot type, and also when the caller's outslot type differs from the scan slot type. Before v3, ExecCopySlot hid that difference. The proposal does not say why the change is safe outside heap.

Thread: [#5 (Bharath Rupireddy, 2026-06-27)](https://www.postgresql.org/message-id/CALj2ACWHZuGSU2m-DkStZYCvv0WxWAzVSnt9w1BZbRzvx0D2xg%40mail.gmail.com)

**Question:** Why does skipping materialization and scanning directly into outslot stay correct for any table AM slot type, including when outslot differs from the AM's own slot type?

Counter-check: held. Message #5 argues only from heap buffer pins, so the request for an argument that holds for any table AM stands. The slot-type part is weak, because RelationFindReplTupleByIndex already scans directly into outslot, and should not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes slot handling in the apply worker's tuple lookup on the subscriber. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The lookup functions work through the table AM API, and the new materialization timing applies to any table AM's slots. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds injection points in execReplication.c and a concurrency TAP test that must stay stable on the buildfarm. |
| performance on the hot path | **yes** | RelationFindReplTupleSeq and RelationFindReplTupleByIndex run for every replicated update and delete; the patch removes a slot copy there. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the tests and injection points be posted as a first patch, with the slot cleanup as a separate second patch?
2. Why is materializing only once before return correct for a table AM whose slots do not hold a buffer pin?
3. What measured cost or reported problem justifies the materialization change beyond consistency between the two functions?
4. How many injection points does v3 add, and what are their names, given that the commit message says one and message #3 says several?
5. How does the TAP test wait for the paused apply worker, and has it run repeatedly without failures on slow machines?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the logical replication apply code and tuple table slot lifetimes across table AMs would have to own it; none takes part in the visible thread.
- **Liability:** The owner signs for the stability of a concurrency TAP test on the buildfarm and, if the cleanup stays, for the correctness of tuple lookup in every apply worker, for as long as the code exists.
- **Author follow-up:** The author answered each review within days and posted v2 and v3, offered to split the patch, and moved the CommitFest entry forward in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is small (about 14 KiB, one backend file and one test) and serves developers of logical replication. The tests can go ahead once split from the cleanup, which needs its own justification.

## Appendix

### Refuted in the counter-check

- `demand-1` Slot cleanup in execReplication.c has no shown benefit. Refuted (not specific): Duplicate of maintenance-fit-1 for the unmeasured cleanup and the split request. Its table AM point is covered by maintenance-fit-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] and [P3] changed the table AM tuple_update and tuple_delete API so it locks updated tuples, and their recorded reasons (wrong RETURNING tuple, API shape) do not apply to apply-worker tests plus slot cleanup in execReplication.c. The thread does not point to an earlier discussion of this idea.
- standards-compat: 0 objection(s). The proposal adds no SQL syntax, GUC, catalog object, psql feature or dump output. It changes no user-visible behaviour: it adds a TAP test, developer-only injection points, and an internal cleanup in execReplication.c. The safety of skipping materialization before the wait and the lock, including for non-heap table access methods, belongs to the design angle.
- replication-ondisk: 0 objection(s). The proposal adds a TAP test, injection points and a slot-handling cleanup inside the apply worker's tuple lookup; it changes no on-disk format, WAL record, catalog, protocol message or extension-facing signature. The rows the subscriber applies stay the same, so this angle has no persistent or cross-node state to object to; the safety of skipping materialization is a code-level question for others.
- security: 0 objection(s). The proposal adds a TAP test, injection points that exist only in builds configured with injection points enabled, and an internal cleanup of slot handling in the apply worker. It adds no SQL surface, privilege, setting or monitoring output, and gives unprivileged users no new action or view. Whether skipping materialization is safe is a data-safety question for the design angle, not a privilege one.
- maintenance-fit: 2 objection(s). The test part alone raises no maintenance objection from this angle. Injection points compile out of production builds, and the patch adds no hot-path cost for non-users.
- already-exists: 0 objection(s). No existing feature, test or extension covers these retry paths; the coverage report cited in message #1 shows them untested. The patch uses injection points and TAP tests, the project's existing tools for this, and adds no second mechanism. [P1] and [P3] concern a table AM API change and do not apply.
- demand: 1 objection(s). The tests have clear developer value: the coverage report shows the retry paths are not exercised, and a test-only injection point costs users little. The objection covers only the bundled cleanup.

### Precedents retrieved

- P1: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - used in the review; reason summarised from its thread
- P2: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - used in the review
- P4: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant; reason summarised from its thread
- P5: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [Persist injection points across server restarts](https://commitfest.postgresql.org/patch/5731/) (cf, Returned with feedback, 2025-07-02) - judged not relevant; reason summarised from its thread
- P7: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P8: [Large files for relations](https://commitfest.postgresql.org/patch/4305/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P10: [add subscription TAP test option "vcregress subscriptioncheck" for MSVC builds](https://commitfest.postgresql.org/patch/3354/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P11: [Backpatching injection point core facilities to REL_17_STABLE](https://commitfest.postgresql.org/patch/5865/) (cf, Withdrawn, 2025-08-17) - judged not relevant
- P12: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - judged not relevant
- P13: [Some refactoring for fixed-numbered stats template in injection_points](https://commitfest.postgresql.org/patch/5187/) (cf, Withdrawn, 2024-10-02) - judged not relevant
- P14: [Replica with large recovery_min_apply_delay is not receiving changes from master after restart until this delay is expired](https://commitfest.postgresql.org/patch/1995/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P15: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P16: [Add more TAP coverage for archive status with crash recovery of standbys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebf6de8692766177a36e7f5fb7545a52a0d5d881) (revert, Partially reverted, 2020-04-27) - judged not relevant
- P17: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - judged not relevant
- P18: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/3536/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P19: [vacuumlo: add test to vacuumlo for test coverage](https://commitfest.postgresql.org/patch/3877/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P20: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P21: [Disable installcheck for tap and regress tests for injection_points module](https://commitfest.postgresql.org/patch/5410/) (cf, Withdrawn, 2025-02-21) - judged not relevant
- P22: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/4160/) (cf, Returned with feedback, 2023-11-11) - judged not relevant
- P23: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P24: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 91 s; cost reported by the CLI: $1.06
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:19:59.959Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
