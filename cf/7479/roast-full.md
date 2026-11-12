# Design pre-review: HashAggregate loses worker spill stats on rescan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7479](https://commitfest.postgresql.org/patch/7479/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [HashAggregate loses worker spill stats on rescan](https://www.postgresql.org/message-id/flat/CAJTYsWW5evhmWDWAgknx%3DGXNMJG%3Dk4-T%2Bd435pZpg8Lqxc%3DkFQ%40mail.gmail.com), started 2026-10-18
- Messages read: 3 of 3
- Patch prose read: v1-0001-Accumulate-HashAggregate-parallel-worker-stats-across-rescans.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The only surviving objection, standards-compat-1, is a fixable scoping question at level 2 and does not stop review. The problem is stated with a reproducer, the fix copies how non-parallel HashAggregate counters behave across rescans, and the cost falls only on EXPLAIN ANALYZE of rescanned parallel plans. A committer who knows executor instrumentation should confirm the merge rule and the stable-branch target.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check that the initial batch counts once in every rescan and worker-reuse case. It also could not check that the patch changes no shared struct, which matters for back branches.

## The proposal, restated

**Problem.** When a Gather or Gather Merge is rescanned, EXPLAIN ANALYZE shows HashAggregate memory, disk and batch stats for parallel workers from the last execution only. A worker that spilled in an earlier scan shows one batch and no disk usage. This affects users who read EXPLAIN ANALYZE output; query results are correct.

**Proposed design.** ExecEndAgg() stops overwriting the shared per-worker instrumentation slot and combines with what earlier workers stored there. Memory and disk usage keep the maximum across executions, and spill batches add up. The initial in-memory batch counts once, to match non-parallel HashAggregate across rescans.

**User-visible surface.**
- EXPLAIN ANALYZE per-worker HashAggregate output (Memory Usage, Disk Usage, Batches) changes for rescanned Gather and Gather Merge.

**What the author claims.**
- ExecEndAgg() overwrites the shared worker slot, so only the last execution is reported.
- The bug dates back to commit 9bdb300dedf.
- Query results are unaffected; this is a reporting bug only.
- hash_batches_used is always at least 1 for hashed and mixed aggregates when ExecEndAgg() records it.
- ExecAggInitializeDSM() zeroes the slot, but a Gather rescan does not, so replacement workers add to the existing count.

**Thread state.** Osama Abdul Qader asked whether the batch-count assumption holds and whether any rescan could count the initial batch twice; Ayush answered both. Osama said he would reproduce and check test coverage, but no review result is visible yet.

## 2. Prior attempts

### Revert "Fix accumulation of parallel worker instrumentation." (similar idea)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAA4eK1LW2aFKzY3=vwvc=t-juzPPVWP2uT1bpx_MeyEqnM+p8g@mail.gmail.com), [https://www.postgresql.org/message-id/flat/20171127175631.GA405@depesz.com](https://www.postgresql.org/message-id/flat/20171127175631.GA405@depesz.com)
- **Date:** decided 2017-12-13; started 2017-12-05
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 2c09a5c12a66087218c7f8cba269cd3de51b9b82. Per further discussion, that doesn't seem to be the best possible fix.
- **Reason (quote):** "Per further discussion, that doesn't seem to be the best possible fix."
- **What is different this time:** P6 changed how generic parallel worker instrumentation accumulates across Gather rescans. This patch changes only the HashAggregate-specific shared stats slot in ExecEndAgg().


## 3. Objections a reviewer is likely to raise

### 1. Rescan semantics fixed for HashAggregate only, other worker-instrumented nodes left unchanged

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

A user runs EXPLAIN ANALYZE on a plan where Gather is rescanned and the workers run both a HashAggregate and a Sort or Memoize. The HashAggregate worker lines would then cover all executions, while the sibling node's worker lines may still cover only the last one. The user compares the two and reaches wrong conclusions about which node spilled.

The patch touches only nodeAgg.c. The proposal does not say whether Sort, Memoize, Hash or IncrementalSort also write per-worker slots that a Gather rescan reuses. If they do, one EXPLAIN output would mix two meanings for worker stats, and with a stable-branch target that mix would ship in minor releases. Matching the non-parallel HashAggregate counters is a sound rule, but a committer would likely ask that it apply to every node with shared worker instrumentation, or that the proposal say why only HashAggregate needs it.

Thread: [#1 (Ayush Tiwari, 2026-10-18)](https://www.postgresql.org/message-id/CAJTYsWW5evhmWDWAgknx%3DGXNMJG%3Dk4-T%2Bd435pZpg8Lqxc%3DkFQ%40mail.gmail.com), [#3 (Ayush Tiwari, 2026-10-20)](https://www.postgresql.org/message-id/CAJTYsWW7DuqJao_sXmXrYh8TpTYQRB6LosN4_XMRjhMY0cM0iw%40mail.gmail.com)

**Question:** Do Sort, Memoize, Hash and IncrementalSort lose per-worker stats on Gather rescan, and if so, why fix only HashAggregate?

Counter-check: held. The patch touches only nodeAgg.c, and the thread does not say whether Sort or Memoize overwrite their worker slots in the same way. Hash already keeps maxima across rescans, so per-node fixes have precedent and the request stays a scoping question.

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
| backpatchability | **yes** | The CommitFest entry targets stable branches, and the fix changes EXPLAIN ANALYZE worker output in minor releases. |
| maintenance burden ("who maintains this in 10 years") | **yes** | ExecEndAgg() gets a HashAggregate-specific merge rule for the shared worker slot that later changes must keep correct. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do Sort, Memoize, Hash and IncrementalSort lose per-worker stats on a Gather rescan, and if so, why does the patch fix only HashAggregate?
2. Which stable branches does the fix target, and is changed EXPLAIN ANALYZE worker output acceptable in a minor release?
3. Can the reproducer from message #1 become a regression test with stable output for parallel HashAggregate rescans?
4. How should a reader relate the combined memory, disk and batch values to the per-loop averages that EXPLAIN shows for the same node?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows parallel executor instrumentation and the HashAggregate spill code. No committer has spoken in the visible thread.
- **Liability:** The owner signs for changed EXPLAIN ANALYZE worker output on every back branch that gets the fix, and for the merge rule staying correct when the HashAggregate counters change.
- **Author follow-up:** The author answered both review questions within a day in message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is a 1.8 KB change to nodeAgg.c that fixes wrong EXPLAIN ANALYZE output for rescanned Gather over HashAggregate. Only users who read such plans benefit, and the remaining point is a scoping question.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Fix covers HashAggregate only, other worker-instrumented nodes left unexamined. Refuted (not specific): Duplicate of standards-compat-1. The added point that new nodes have no reminder to add a merge rule applies to all shared worker instrumentation, which this patch does not introduce.

### Review angles

- prior-attempts: 0 objection(s). P6 is the only close relative. Its recorded reason says the earlier fix was not the best one but does not say why, so it gives no failure scenario that applies here. P12 is about Gather shutting down early under Limit and destroying the parallel context, which this patch does not do. The thread does not point to any earlier discussion of the same idea.
- standards-compat: 1 objection(s). The only user-facing change is EXPLAIN ANALYZE output, and the patch copies how non-parallel HashAggregate counters already behave across rescans. Neither precedent's recorded reason applies: P6 gives no specific reason, and P12 is about early shutdown destroying the parallel context.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state: it changes how ExecEndAgg() fills a transient shared-memory instrumentation slot used only for EXPLAIN ANALYZE, with no on-disk, WAL, catalog, protocol or struct change. Only nodeAgg.c is touched, so a stable-branch backpatch should not raise ABI concerns, though this cannot be confirmed without the code.
- security: 0 objection(s). The patch changes only how EXPLAIN ANALYZE combines HashAggregate memory, disk and batch stats from parallel workers across Gather rescans. Only the user who runs the query can see these stats, and the patch adds no new privilege, data exposure, escalation path or unbounded resource use, so there is no security angle to object to.
- maintenance-fit: 1 objection(s). From the maintenance angle this is a small, contained bugfix. ExecEndAgg runs once per worker, so the patch adds no per-tuple, per-query or per-backend cost, and its semantics follow how non-parallel HashAggregate counters already behave across rescans. The one concern above is about consistency with other nodes, not about the fix itself.
- already-exists: 0 objection(s). This is a bug fix to EXPLAIN ANALYZE reporting inside nodeAgg.c. No existing feature, setting or extension can correct per-worker HashAggregate stats that ExecEndAgg() overwrites on a Gather rescan, so this angle finds no alternative. [P6] and [P12] concern other instrumentation fixes, and their recorded reasons do not apply here.
- demand: 0 objection(s). This is a bugfix to EXPLAIN ANALYZE output that is wrong today for any rescanned Gather or Gather Merge over HashAggregate. A one-file change has no cost for non-users, and Postgres does not require demand evidence to fix wrong output. The design questions, such as how to combine values across rescans and whether Sort, Memoize and Hash have the same overwrite problem, belong to other angles.

### Precedents retrieved

- P1: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [Allow setting parallel_workers on partitioned tables](https://commitfest.postgresql.org/patch/2987/) (cf, Returned with feedback, 2021-09-04) - judged not relevant; reason summarised from its thread
- P3: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P4: [speeding up GIN build with parallel workers](https://commitfest.postgresql.org/patch/527/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P5: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P6: [Revert "Fix accumulation of parallel worker instrumentation."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1d6fb35ad62968c8678d0321887e2b9ca8fe1a84) (revert, Reverted, 2017-12-13) - used in the review
- P7: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant; reason summarised from its thread
- P8: [improve DEBUG1 logging of parallel workers for CREATE INDEX](https://commitfest.postgresql.org/patch/5490/) (cf, Withdrawn, 2025-02-03) - judged not relevant; reason summarised from its thread
- P9: [system load consideration hook for parallel workers](https://commitfest.postgresql.org/patch/725/) (cf, Rejected, 2016-09-22) - judged not relevant; reason summarised from its thread
- P10: [Add a max_parallel_workers GUC](https://commitfest.postgresql.org/patch/703/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P11: [logicalrep_worker_launch - counting/checking the type limits](https://commitfest.postgresql.org/patch/4499/) (cf, Withdrawn, 2024-04-23) - judged not relevant; reason summarised from its thread
- P12: [Don't shut down Gather\[Merge\] early under Limit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=080313f8296fb0bcc74bd70fc8e15cd64f45945e) (revert, Partially reverted, 2019-11-26) - used in the review
- P13: [explain analyze rows=%.0f](https://commitfest.postgresql.org/patch/3704/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P14: [Revert "Fix race in Parallel Hash Join batch cleanup."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7f7f25f15edb6eacec58179ef5285e874aa4435b) (revert, Reverted, 2021-03-17) - judged not relevant
- P15: [Improvements of worker_spi.c example](https://commitfest.postgresql.org/patch/4315/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P16: [Undo worker and transaction rollback](https://commitfest.postgresql.org/patch/1828/) (cf, Withdrawn, 2019-09-09) - judged not relevant
- P17: [bloom filter in hash joins with batching](https://commitfest.postgresql.org/patch/446/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P18: [Improve logicalrep_worker_launch() logic](https://commitfest.postgresql.org/patch/5734/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P19: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant
- P20: [Sampling-based timing for EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/4100/) (cf, Returned with feedback, 2023-07-03) - judged not relevant
- P21: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P22: [Revert "Use CP_SMALL_TLIST for hash aggregate"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0babd109801e5ecd90df29589c23c6daf3ae69f7) (revert, Reverted, 2020-07-13) - judged not relevant
- P23: [Connection string usage for Core Postgresql client applications](https://commitfest.postgresql.org/patch/2354/) (cf, Withdrawn, 2020-01-21) - judged not relevant
- P24: [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://commitfest.postgresql.org/patch/3751/) (cf, Returned with feedback, 2024-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 72 s; cost reported by the CLI: $0.74
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 54 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:59:50.885Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
