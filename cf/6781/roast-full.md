# Design pre-review: pgstat: Flush some statistics within running transactions, take 2

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6781](https://commitfest.postgresql.org/patch/6781/) (Needs review); authors: Sami Imseih (simseih)
- Thread: [pgstat: Flush some statistics within running transactions, take 2](https://www.postgresql.org/message-id/flat/CAA5RZ0uA-4qcD3%2B2hjcE_-zQUBhvWf5foPM2vzYneFKrJLsBDQ%40mail.gmail.com), started 2026-05-17
- Messages read: 44 of 44
- Patch prose read: v8-0001-pgstat-Prepare-db-pending-stats-before-flush.patch, v8-0002-pgstat-Split-table-stat-counters-into-transaction.patch, v8-0003-pgstat-Allow-pg_stat_force_next_flush-to-work-in-.patch, v1-0001-pgstat-Introduce-pg_stat_report_anytime-for-mid-t.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection is fatal, and none is at level 1. Two fixable level 2 points remain. replication-ondisk-1: a custom stats callback ported with the smallest change double counts without any error. standards-compat-2: the existing pg_stat_force_next_flush() takes on immediate-flush behaviour under a name that says "next". A committer who knows pgstat has to decide both; neither stops the idea.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The retained-entry and baseline logic, the requeue of entries during a pass, and error safety of a flush inside a transaction could not be judged from the prose.

## The proposal, restated

**Problem.** A backend publishes its pending statistics only at transaction end, so a long transaction stays invisible in the cumulative statistics views until it commits or aborts. The author needs an on-demand mid-transaction flush for extensions, first for planned pg_stat_statements scalability work, and for users and tests that want current numbers.

**Proposed design.** pg_stat_force_next_flush() flushes immediately when called in a transaction, and a C API does the same for the calling backend. Non-transactional counters such as scans and blocks hit are flushed at once, while tuple insert, update and delete counters wait for the transaction boundary. Pending entries stay allocated until transaction end with a flushed baseline, so each flush publishes only the delta, and entries that receive new counters during a pass are requeued.

**User-visible surface.**
- pg_stat_force_next_flush() flushes immediately inside a transaction, becomes PARALLEL UNSAFE, and is now documented (catversion bump).
- last_seq_scan and last_idx_scan use the statement start time when flushed during a transaction.
- flush_pending_cb takes a new xact_boundary argument and returns a PgStat_FlushResult enum (LOCK_CONFLICT, DONE, PARTIAL) instead of bool.
- flush_static_cb takes a transaction-boundary flag.
- Custom stats callbacks must not merge retained pending data twice and must not call pgstat_force_next_flush().

**What the author claims.**
- Only callers that force a flush mid-transaction pay any cost, and an unchanged retained entry costs a memcmp, not a lock.
- A benchmark with up to 512 relations and 256 clients showed no difference beyond about 1% tps noise for the per-pass reset scan.
- Throttling belongs to the caller, so pgstat needs no rate limit.
- The change is needed for the pg_stat_statements scalability patches in another thread.

**Thread state.** Bertrand Drouvot reviewed many versions and called v10 in good shape, and Michael Paquier committed the counter split refactoring and the documentation of pg_stat_force_next_flush(). The main patch, now v18, gets only rebases with no new review since early September.

## 2. Prior attempts

### Flush some statistics within running transactions (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6389/](https://commitfest.postgresql.org/patch/6389/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal](https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal)
- **Date:** decided 2026-03-25; started 2026-01-12
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Michael Paquier objected to the timeout design after the entry was marked Ready for Committer. He said it requires SIGALRM handlers and timeouts in many processes that never used them, and pgstat_schedule_anytime_update() calls in stats kinds and extensions. He proposed instead a procsignal-based client API (pg_stat_report()) with strategic flush calls, and opposed removing existing report calls. Sami Imseih's later idle-in-transaction timeout design did not help a single long analytical query, so no design reached agreement.
- **Reason (quote):** "This design requires enabling a new signal with a signal handler in a lot of processes that did not do that. Enabling timeouts in a bunch of new processes, while claiming it is fine to do, sounds like something we should rather be careful about." ([message](https://www.postgresql.org/message-id/aZ-_AkG3cmeTbdFn%40paquier.xyz))
- **What is different this time:** P1 used a periodic timeout and later a stats_flush_interval GUC across backends and auxiliary processes. This proposal has no timer, no new signal handler and no GUC: the calling backend flushes on demand, which is the direction the P1 objector suggested according to message #1.


## 3. Objections a reviewer is likely to raise

### 1. Custom stats callbacks can silently double count after the API change

fixable · level 2, rubric item 9 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

An extension with a custom variable-numbered kind is ported by adding the xact_boundary argument and returning PGSTAT_FLUSH_DONE. When a user calls pg_stat_force_next_flush() inside a transaction, the retained pending entry is merged again at the next flush or at commit. The inflated counters reach shared memory and the stats file written at shutdown, and nothing reports an error.

The patch keeps pending entries allocated mid-transaction (message #4), so every callback must now clear or baseline what it published. The rule exists only as a comment on flush_pending_cb (message #12). The signature change makes the build fail, but the minimal port that fixes the build is the one that double counts. Message #3 reordered the enum to avoid silent misbehavior in callbacks, but the retained-pending rule has no such guard.

Thread: [#3 (Bertrand Drouvot, 2026-07-31)](https://www.postgresql.org/message-id/amyuOVEGvwOs5J0Y%40bdtpg), [#4 (Sami Imseih, 2026-07-31)](https://www.postgresql.org/message-id/CAA5RZ0v8REO_RHHBcTPdmoO6a2fkT%3Dxp4iKG-%3DLh_FvmG%3DSzgw%40mail.gmail.com), [#9 (Bertrand Drouvot, 2026-08-06)](https://www.postgresql.org/message-id/anSCUOyzesTT%2FqwR%40bdtpg), [#12 (Sami Imseih, 2026-08-14)](https://www.postgresql.org/message-id/CAA5RZ0u%2Bmb3H7in2%3DZjZE4P6Aujm%3Doph1KOTQLtoOZD2phyOvw%40mail.gmail.com)

**Question:** What makes a ported custom callback that does not clear or baseline its pending data fail visibly, for example in an assert-enabled build, rather than double count?

Counter-check: held. Message #12 confirms the no-double-merge rule exists only as a comment on flush_pending_cb, and the proposal has no check that makes a violation fail. The test module shows the pattern, but a minimal port that returns PGSTAT_FLUSH_DONE still double counts without notice.

### 2. Existing function and C API get new semantics under the old name

fixable · level 2, rubric item 12 · design, ABI / extension API, SQL-standard conflict or non-standard syntax · `standards-compat-2`

An extension or test harness already calls pgstat_force_next_flush() or pg_stat_force_next_flush() inside a transaction, from a hook or from a function, expecting only a deferred flush. After upgrade, the call runs every flush callback on the spot, takes shared stats locks, and can raise an ERROR there. The caller sees this only when it fails.

v1 added a separate pg_stat_report_anytime(). v2 moved the immediate flush into the existing function, whose name still says the flush happens at the next occasion (message #2). Michael documented the function as end-of-transaction and test-oriented in message #36, and v18 changes that documented contract in the same cycle. The function also changes from PARALLEL RESTRICTED to PARALLEL UNSAFE, which can change plans for queries that already call it.

Thread: [#1 (Sami Imseih, 2026-05-17)](https://www.postgresql.org/message-id/CAA5RZ0uA-4qcD3%2B2hjcE_-zQUBhvWf5foPM2vzYneFKrJLsBDQ%40mail.gmail.com), [#2 (Sami Imseih, 2026-07-30)](https://www.postgresql.org/message-id/CAA5RZ0u84eMFeFWMaEo0D84ed3jF_RY0%3DRWD8tDwQWsNsU1qvA%40mail.gmail.com), [#8 (Sami Imseih, 2026-08-06)](https://www.postgresql.org/message-id/CAA5RZ0uqJYn9SXeZuA_pmr1E5xpTV45Sw7TX8pqHj44o_0BY4w%40mail.gmail.com), [#36 (Michael Paquier, 2026-09-09)](https://www.postgresql.org/message-id/aqEONVuLhgBiqHW6%40paquier.xyz), [#39 (Michael Paquier, 2026-09-10)](https://www.postgresql.org/message-id/aqIxIveLyADWOwlb%40paquier.xyz)

**Question:** Why should the existing deferred-flush function take on immediate-flush semantics rather than a new function or an option, given current C and SQL callers?

Counter-check: held. The thread gives no reason for moving the immediate flush into a function whose name says "next", so the consistency question stands. The documented-contract point is weak: that text was added this cycle and message #36 expects to update it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | pg_proc.dat changes pg_stat_force_next_flush() to PARALLEL UNSAFE and needs a catversion bump. |
| ABI / extension API | **yes** | flush_pending_cb gets an xact_boundary argument and an enum return type; flush_static_cb gets a transaction-boundary flag. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: Existing function and C API get new semantics under the old name |
| new GUC | no |  |
| planner behaviour change | **yes** | The PARALLEL UNSAFE marking stops parallel plans for queries that call pg_stat_force_next_flush(). |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Relation, index and function stats keep a flushed baseline with delta logic, and custom kinds carry a new no-double-merge rule. |
| performance on the hot path | **yes** | The patch touches the pending-stats flush path that every backend runs at transaction end. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What makes a ported custom callback that returns PGSTAT_FLUSH_DONE without clearing or baselining its pending data fail visibly, for example in an assert-enabled build?
2. Why does the existing pg_stat_force_next_flush() take on the immediate flush, instead of a new function or an option as in v1?
3. Where is the rule that custom flush callbacks must avoid cyclic dependencies documented for extension authors?
4. What does one transaction cost when it forces many flushes over hundreds of retained entries?
5. Is dropping the flush of another backend by PID, including auxiliary processes, final for this patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier committed the counter split and the documentation of pg_stat_force_next_flush() from this thread. The thread does not say whether he accepts the main patch design.
- **Liability:** The owner signs for the retained-pending and baseline logic in relation, index and function stats, and for the changed callback contract of custom stats kinds, for as long as custom kinds exist.
- **Author follow-up:** The author has posted 18 versions, adopted review feedback, and keeps rebasing, most recently moving the entry to the next CommitFest on 2026-11-10.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The patch is a prerequisite for the pg_stat_statements scalability work and serves users and tests that want current numbers from a long transaction. The main patch is about 98 KiB across all pgstat kinds and has had no new review since early September, so it needs a committer's design decision on the two open points.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Unchanged custom stats kinds can double count after a mid-transaction flush. Refuted (wrong facts): The patch changes flush_pending_cb to take a third argument and return an enum, so an old two-argument bool callback gets an incompatible-pointer-type diagnostic, an error by default in current GCC and Clang. The ported-callback case is covered by replication-ondisk-1.
- `maintenance-fit-1` New flush callback contract for custom stats kinds fails silently. Refuted (not specific): Duplicate of replication-ondisk-1. Its extra point about calling pgstat_force_next_flush() from a callback is already guarded by pgStatFlushInProgress since message #12.
- `maintenance-fit-2` Every future relation counter needs a correct transactional classification. Refuted (no failure scenario): The scenario depends on a mistake in a future patch. The committed split puts transactional counters in their own struct, so a new counter needs an explicit choice of group that a reviewer can see.
- `maintenance-fit-3` Repeated in-transaction flushes rescan all retained entries. Refuted (wrong facts): The example in message #2 is throttled to once every 10 seconds and is not part of the patch. Message #12 says pg_stat_statements flushes only in the backend that reads the view, so the per-statement flush scenario does not follow from the material.
- `demand-1` Patch as posted does not let anyone see another session's long transaction. Refuted (already handled): Message #1 presents the patch as an on-demand API, the direction Michael Paquier suggested, with pg_stat_statements as the named consumer. [P1] failed over its timeout design and shows that a second author wanted this visibility, so it does not support the objection.

### Review angles

- prior-attempts: 0 objection(s). The one close precedent, [P1], died on its timeout and signal design, and this proposal drops both; v1's signal-by-PID path is gone too. Michael Paquier committed the prerequisite refactoring and the documentation patch (messages #40, #39), so the old reason does not carry over. Two points stay open for other angles. One is whether a self-only flush serves sessions watching a long transaction from outside. The other is whether the statement-boundary follow-up in message #2 brings back P1's timing questions.
- standards-compat: 2 objection(s). pg_dump, psql and GUC concerns do not apply. The change of last_seq_scan and last_idx_scan to statement start time is documented and has a small effect, so it is not raised here.
- replication-ondisk: 1 objection(s). The proposal changes no on-disk format, WAL, replication or wire protocol. The catalog change is a proparallel flip with a catversion bump. The only cross-boundary concern is the custom stats callback contract.
- security: 0 objection(s). The current design flushes only the calling backend's own pending statistics into views that are already visible, so it gives no new access to other sessions or data. Cross-backend flushing by PID was dropped after v1. The material does not show whether pg_stat_force_next_flush() is granted to PUBLIC. It also has no measurement of a looped in-transaction flush contending on shared stats entry locks, but no concrete harm beyond what repeated short transactions already allow.
- maintenance-fit: 3 objection(s). The goal is useful, a committer helped shape the design, and preparatory pieces are committed. The objections are about long-run contracts and cost, which a new version could address.
- already-exists: 0 objection(s). No existing feature gives other sessions a long transaction's counters before it ends. pg_stat_xact_* shows them only to the session that owns the transaction, and pg_stat_force_next_flush() waits for the transaction boundary. An extension cannot flush retained pending entries without core changes. The earlier attempt [P1] tried a timer-based design, and this proposal follows the on-demand API suggested there, so it does not duplicate an existing mechanism.
- demand: 1 objection(s). Demand for mid-transaction stats visibility is shown by [P1] and by Michael's test use (message #36). The objection concerns whether this step reaches those users.

### Precedents retrieved

- P1: [Flush some statistics within running transactions](https://commitfest.postgresql.org/patch/6389/) (cf, Returned with feedback, 2026-03-25) - used in the review; reason summarised from its thread
- P2: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P3: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P4: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P5: [Fix inconsistency in reporting checkpointer stats](https://commitfest.postgresql.org/patch/4078/) (cf, Returned with feedback, 2024-10-10) - judged not relevant; reason summarised from its thread
- P6: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - used in the review; reason summarised from its thread
- P7: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P8: [explain to report bucket and memory stats for TupleHashTable nodes](https://commitfest.postgresql.org/patch/2428/) (cf, Returned with feedback, 2020-07-23) - judged not relevant; reason summarised from its thread
- P9: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant; reason summarised from its thread
- P10: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - judged not relevant; reason summarised from its thread
- P11: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant
- P12: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P13: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P14: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant
- P15: [Revert "Flush Memoize cache when non-key parameters change"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dad20ad4709f602b4827a1ab2b0e715f36c548c3) (revert, Reverted, 2021-11-24) - judged not relevant
- P16: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P17: [Revert function to get memory context stats for processes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fb844b9f06568aa29303122ac0bb7b4c3cce2542) (revert, Reverted, 2025-05-23) - judged not relevant
- P18: [Revert "Add some temporary instrumentation to pgstat.c."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5a2bb060121f574d6587e5fc6c7f1d4e3a759351) (revert, Reverted, 2012-05-15) - judged not relevant
- P19: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P21: [Sampling in pg_stat_statements](https://commitfest.postgresql.org/patch/5390/) (cf, Returned with feedback, 2025-04-09) - judged not relevant
- P22: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P23: [pg_stat_statements with plans](https://commitfest.postgresql.org/patch/969/) (cf, Returned with feedback, 2017-03-04) - judged not relevant
- P24: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 20; wall time: 251 s; cost reported by the CLI: $5.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 7 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 84 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:00:12.113Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
