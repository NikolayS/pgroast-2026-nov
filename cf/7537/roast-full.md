# Design pre-review: Gather rescan keeps the first scan's tuple bound in workers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7537](https://commitfest.postgresql.org/patch/7537/) (Needs review); authors: shihao zhong (zhong950419)
- Thread: [bug: Gather rescan keeps the first scan's tuple bound in workers](https://www.postgresql.org/message-id/flat/2701c83e33f3fe66320291fb5d1b74ca2cb90bd7.camel%40j-davis.com), started 2026-09-18
- Messages read: 2 of 2
- Patch prose read: v1-0001-Pass-the-current-tuple-bound-to-workers-on-Gather.patch, v1-0002-Add-test-for-Gather-rescan-with-a-changing-tuple-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is a confirmed wrong-results bug with a default-settings repro in message #1, and the fix follows what ExecInitParallelPlan() already does. The only surviving objection is standards-compat-1: the patch says Backpatch-through: 14 and changes an exported function signature in back branches. That is a fixable choice, and message #2 already names the alternative. No level fails.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that both Gather and Gather Merge pass the current bound at the right point on rescan. It also could not check whether the test output is deterministic.

## The proposal, restated

**Problem.** When a Limit with a parameter-dependent count sits over Gather or Gather Merge and the node is rescanned, workers keep the tuple bound from the first scan. If a later scan needs more rows, the query silently returns too few rows, or with Gather Merge over a Sort the right number of wrong rows. The report says all supported branches are affected, back to Postgres 11.

**Proposed design.** ExecParallelReinitialize() gets a tuples_needed argument and writes it into the shared executor state in the DSM segment, as ExecInitParallelPlan() already does. Gather and Gather Merge pass their current bound on rescan. A second, optional patch adds a regression test with the leader out of the scan.

**User-visible surface.**
- C API: ExecParallelReinitialize() gains a tuples_needed argument, which changes an exported function signature in back branches.

**What the author claims.**
- The bug is a wrong-results bug in all supported branches, introduced by commit 3452dc5240d in Postgres 11.
- A bound that shrinks, or an unbounded first scan, is harmless and only loses the early exit.
- The reporter's repro fails at default settings; with parallel_leader_participation = off the result is stable.
- The patch author can reproduce it on master with a 1000-row table.
- The patch author knows of no extension that calls ExecParallelReinitialize().

**Thread state.** Jeff Davis reported the bug and confirmed it, and shihao zhong replied with a fix and a test. Nobody has reviewed the patch, and the back-branch ABI question is open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-branch fix changes the signature of an exported executor function

fixable · level 2, rubric item 11 · backpatchability, ABI / extension API · `standards-compat-1`

An extension built against an earlier 14-18 minor release calls ExecParallelReinitialize() with the old argument list. After a minor upgrade the server reads an undefined tuples_needed value. The workers then stop at a random bound, or the extension fails to build, and the user finds out only after the upgrade.

Patch 0001 says "Backpatch-through: 14" and adds a parameter to a function declared in execParallel.h. Minor releases normally keep exported signatures stable, so an extension does not need a rebuild. In message #2 the author says no caller is known and offers a new function as the alternative, but the thread does not choose. An absence of known callers is weak evidence for out-of-tree code that uses parallel contexts.

Thread: [#2 (shihao zhong, 2026-09-19)](https://www.postgresql.org/message-id/CAGRkXqRGE%2BTwD0Ttmb4b1J8Vy-drrkZYhh6hOf2_Uz5XKtXsEA%40mail.gmail.com)

**Question:** Will the back-branch versions keep the existing ExecParallelReinitialize() signature, for example by setting the bound through a separate function, and leave the new argument only on master?

Counter-check: held. The commit message says "Backpatch-through: 14" and the patch changes a prototype in execParallel.h. Message #2 names the ABI-compatible alternative but the thread does not choose it, so the question is open and fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | ExecParallelReinitialize() in execParallel.h gains a tuples_needed argument. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Patch 0001 says Backpatch-through: 14 and changes an exported signature in those branches. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the back-branch versions keep the existing ExecParallelReinitialize() signature, for example with a separate function that sets the bound, and change the signature only on master?
2. Is the 0002 regression test stable across platforms and repeated runs, given that it depends on which rows the workers read with the leader out of the scan?
3. Does the fix also cover the plain Gather case from message #1, and does the test exercise it or only Gather Merge?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Jeff Davis reported and confirmed the bug in message #1 but has not reviewed the patch. The owner needs to know the parallel executor's DSM reuse on rescan.
- **Liability:** A back-patched executor change in all supported branches, including the ABI choice for ExecParallelReinitialize() and any buildfarm instability from the new test.
- **Author follow-up:** The patch author offered in message #2 to switch to a new function for back branches if the signature change is a concern.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The query returns wrong results without an error at default settings in all supported branches, and the fix is about 4 KiB in three executor files and one header. The plan shape needs a parameter-dependent Limit over a rescanned Gather or Gather Merge, so not every user hits it.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-branch fix changes the signature of an exported executor function. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-1` Back-branch signature change of ExecParallelReinitialize() is left undecided. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-2` Per-scan DSM fields still have two write paths that can drift. Refuted (no failure scenario): The scenario needs a future patch to add a per-scan field, so it is not a failure of this fix. The struct's other fields are param_exec, which ExecParallelReinitialize() already rewrites, and eflags and jit_flags, which do not change between scans.

### Review angles

- prior-attempts: 0 objection(s). No precedent found that tried this fix or rejected it for a reason that applies here. [P1] reverted an early Gather shutdown that broke rescans, and [P4] restored a rescan test; neither recorded reason applies to restoring the tuple bound on rescan, and [P20] has no recorded reason.
- standards-compat: 1 objection(s). Apart from the back-branch ABI point, the fix adds no syntax, GUC, catalog or dump change, and users see only corrected query results.
- replication-ondisk: 1 objection(s). The fix adds no on-disk, WAL, catalog or wire-protocol state. The bound lives only in the per-query DSM segment, which is created and freed with the query, so the only issue for this angle is back-branch ABI.
- security: 0 objection(s). The fix has no privilege or exposure aspect. It rewrites one field in the shared executor state that only the leader writes and only its own workers read, during a rescan the user already triggers. It adds no new SQL surface, setting, role or way to see other sessions' data, and gives an unprivileged user no new way to use resources. The wrong results come from the existing bug, not from the fix, and a stale bound that is too large only loses the early exit.
- maintenance-fit: 2 objection(s). The fix itself is not in doubt: it is a confirmed wrong-results bug, the change adds no cost for non-parallel or non-rescan plans, and it fits core. The objections above concern how the fix is backpatched and how drift is prevented.
- already-exists: 0 objection(s). This is a wrong-results bug fix in the executor's Gather rescan path, and no existing feature, setting or extension can give workers the current tuple bound after a rescan. Turning off parallel query or parallel_leader_participation hides or changes the symptom but does not fix it, so the already-exists angle has nothing to raise.
- demand: 0 objection(s). This is a wrong-results bugfix, so demand is not in question. The reproduction uses default settings and plain LATERAL with a parameterized LIMIT, and a committer confirmed it in message #1. Every user who hits this plan shape gets silently wrong answers. The fix costs nothing for users who do not hit it. Back-branch ABI handling is a design question for another angle.

### Precedents retrieved

- P1: [Don't shut down Gather\[Merge\] early under Limit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=080313f8296fb0bcc74bd70fc8e15cd64f45945e) (revert, Partially reverted, 2019-11-26) - used in the review
- P2: [Fix comment about generate_gather_paths](https://commitfest.postgresql.org/patch/2876/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P3: [Fix the statistics gathering in case of commit and chain](https://commitfest.postgresql.org/patch/3657/) (cf, Returned with feedback, 2022-10-06) - judged not relevant; reason summarised from its thread
- P4: [Restore test case from a2b70c89ca1a5fcf6181d3c777d82e7b83d2de1b.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6c2c5bea3cec4c874d1ee225bb6e222055c03d75) (revert, Partially reverted, 2017-08-30) - used in the review
- P5: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant; reason summarised from its thread
- P6: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P7: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P8: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P9: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P10: [Disable parallel query by default](https://commitfest.postgresql.org/patch/5751/) (cf, Rejected, 2026-02-11) - judged not relevant; reason summarised from its thread
- P11: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P12: [Revert "Allow parallel workers to cope with a newly-created session user ID."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e6a9637488e2673efb87f8ead657789e9889fb17) (revert, Reverted, 2024-08-01) - judged not relevant
- P13: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P14: [Add the ability to limit the amount of memory that can be allocated to backends.](https://commitfest.postgresql.org/patch/3867/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P15: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant
- P16: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P19: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant
- P20: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - used in the review
- P21: [Reuse data from readRecordBuf in XLogDecodeNextRecord](https://commitfest.postgresql.org/patch/6343/) (cf, Withdrawn, 2026-02-09) - judged not relevant
- P22: [pg_dump new feature: exporting functions only](https://commitfest.postgresql.org/patch/3051/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P23: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P24: [Revert "TupleHashTable: store additional data along with tuple."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4a07f532b40a64fb4714a3f7ab6063435411edb) (revert, Reverted, 2025-01-13) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 110 s; cost reported by the CLI: $0.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 62 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:02.022Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
