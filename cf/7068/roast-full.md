# Design pre-review: Lost KEY SHARE lock information during toasting allows foreign-key violation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7068](https://commitfest.postgresql.org/patch/7068/) (Needs review); authors: Michael Paquier (michael-kun)
- Thread: [lost lock during toasting allows fk violation](https://www.postgresql.org/message-id/flat/CA%2BCOZaC-2-GQabq%2Bem11PVdAX6H%2BvFYvU6mWC%3DQyNHb1RBuczg%40mail.gmail.com), started 2026-07-28
- Messages read: 4 of 4
- Patch prose read: 0001-Fix-loss-of-tuple-lock-during-TOAST-step-in-heap_upd.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

The bug is real and the fix is wanted: message #1 shows an orphaned child row from ordinary DML. Two fixable design points survive at level 2. replication-ondisk-1 shows v1 writes a lock-only xmax naming the updater and a duplicate multixact member, and maintenance-fit-1 shows the recheck condition already missed one unlock path. Rui's follow-up patches address both, but no merged version exists and nobody has answered message #4.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the handling of locker_remains and checked_lockers, the page-extension path and the exact headers written by the merged version could not be checked. The end of message #4 is cut off.

## The proposal, restated

**Problem.** heap_update() releases the buffer lock for TOAST or page-extension work, and a KEY SHARE lock taken in that window by a foreign-key check is overwritten when the update stamps xmax. The referenced row can then be deleted without waiting, leaving an orphaned child row. The reporter reproduced it on master with plain large TOAST values, in 12 of 12 timed trials.

**Proposed design.** v1 moves the computation of the old and new tuple xmax to after the TOAST and page-extension block, once the buffer lock is held again. If the xmax or t_infomask changed meanwhile, the patch assumes a concurrent locker and sets locker_remains. A reviewer's follow-up patches gate the recheck on the condition that released the buffer lock and keep the updater's temporary lock out of both tuples.

**User-visible surface.**
- New isolation test fk-toast-lock (v1).
- New injection point heap_update-toast-unlocked, plus an isolation test, in the reviewer's 0002.

**What the author claims.**
- The bug is an ordering problem in heap_update(): xmax is computed before the TOAST step and written after it.
- An isolation test that renames a TOAST table triggers the bug without a concurrent workload.
- fk-toast-lock fails on master and passes with v1 (reviewer's report).
- The reviewer says the orphan is still reachable with v1, through an update of an unrelated column when the toaster has nothing to do.
- The reviewer says v1 leaves the new tuple locked by the updater itself and adds the updater to the multixact twice; their 0001 fixes both.

**Thread state.** The reporter, Michael Paquier (two messages and v1) and reviewer Rui Zhao (0001 and 0002 on top of v1) took part; Peter Geoghegan and Alvaro were CC'd but did not reply. The CommitFest entry needs a rebase, and nobody has answered Rui's findings.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. v1 persists wrong xmax and multixact state on disk and on standbys

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade, backpatchability · `replication-ondisk-1`

A minor release ships v1. A plain update of a row with an external value stamps the new tuple with a lock-only key-share xmax naming the updater itself. Updates that meet a real key-share locker create multixacts holding the updater twice. Both forms are WAL-logged, replayed on standbys and kept through pg_upgrade, and nobody notices until the headers confuse a later lock check or pg_multixact members grow faster.

Message #4 shows the new tuple's xmax at the updater's own xid on v1, where master writes 0. It also shows an extra fornokeyupd member in both multixacts, and a caller's tuple whose header no longer matches the stored one. These headers live in heap pages and pg_multixact and pass to standbys through the update WAL record, so a stable-branch fix cannot take them back once released. Nobody has answered message #4, and message #3 says the locker_remains logic is unsettled.

Thread: [#3 (Michael Paquier, 2026-07-29)](https://www.postgresql.org/message-id/aml62IuY_wUS5zvi%40paquier.xyz), [#4 (Rui Zhao, 2026-08-05)](https://www.postgresql.org/message-id/CAHWVJhGtxkEhiKU7ntFGQ2Lw9kUTebJD3N-xR8Q9OXem37A%2BUg%40mail.gmail.com)

**Question:** Which xmax and infomask invariants does the merged version keep on the old and new tuple, and does replay produce the same headers as the primary?

Counter-check: held. Message #4 shows v1 writing the updater's own xid as a lock-only xmax and a duplicate multixact member. Rui's 0001 restores the master output, but no merged version exists and nobody has answered, so this stays open as fixable.

### 2. Recheck is tied to a proxy condition, so later unlock paths can skip it

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds another path in heap_update() that drops the buffer lock, or changes when the toaster returns its input tuple. The xmax recheck does not run on that path, and a concurrent KEY SHARE lock is lost again. Nothing fails until someone finds an orphaned child row in production.

Message #4 shows the v1 recheck already misses one unlock path. v1 gates on `heaptup != newtup || newbuf != buffer`, but the unlock happens under `need_toast || newtupsize > pagefree`. Rui's 0001 realigns the two conditions, yet the design still needs every future edit of this block to keep the conditions in step. Only an injection-point test for one path would fail loudly if they drift.

Thread: [#3 (Michael Paquier, 2026-07-29)](https://www.postgresql.org/message-id/aml62IuY_wUS5zvi%40paquier.xyz), [#4 (Rui Zhao, 2026-08-05)](https://www.postgresql.org/message-id/CAHWVJhGtxkEhiKU7ntFGQ2Lw9kUTebJD3N-xR8Q9OXem37A%2BUg%40mail.gmail.com)

**Question:** Can the recheck be keyed on a single record that the buffer lock was released, so any new unlock path in heap_update() is covered without a separate change?

Counter-check: held. Message #4 shows the two conditions already drifted once inside v1. Rui's 0001 realigns them, but the material does not show that one record of the unlock drives the recheck, so the question is fair.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch changes which xmax and infomask values heap_update() stamps on heap tuples, and which multixact members it creates. |
| WAL / replication / standby | **yes** | The tuple headers written by heap_update() reach standbys through the update WAL record; no new record type is added. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest target is stable, but the reporter tested master only and the affected back branches are not listed. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The xmax recheck in heap_update() must stay in step with every path that releases the buffer lock. |
| performance on the hot path | **yes** | The patch reorders work inside heap_update(); the extra recheck runs only when the buffer lock was released. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which xmax and infomask values does the merged version write on the old and new tuple, and does replay on a standby produce the same headers?
2. Can the recheck be keyed on one record that heap_update() released the buffer lock, so a new unlock path is covered without a separate change?
3. Which back branches have the bug, and does the same patch apply to each of them?
4. Does page extension without TOAST work open the same window, and does a test cover that path?
5. How should locker_remains and checked_lockers be set when the xmax changed while the buffer lock was released?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier wrote v1 and asked for more review; a committer who knows heapam tuple locking and multixacts should also confirm the merged version.
- **Liability:** The owner signs for the tuple xmax and multixact state that heap_update() writes in every back-patched branch. Wrong headers persist in heap pages and on standbys after a minor release and cannot be taken back.
- **Author follow-up:** Michael posted v1 one day after the report and invited comments, but nobody has answered Rui's findings in message #4 and the entry needs a rebase.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Users of foreign keys whose referenced tables have TOASTed columns can get orphaned child rows from ordinary DML (message #1). The patch is small, but it changes tuple locking in heap_update() for stable branches, so review needs care and one merged, rebased version.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Stable-branch change to heap_update locking has no engaged heapam owner. Refuted (not specific): Duplicate of replication-ondisk-1 for the failure scenario. The rest is committability: a committer wrote v1 and asked for more review, and missing sign-off on an unmerged patch is not a design flaw.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The precedent search returned no row about the same or a similar idea, and the thread points to no earlier discussion of losing a KEY SHARE lock during the TOAST step of heap_update(). This is a bug fix, so a prior-rejection objection does not apply.
- standards-compat: 0 objection(s). This is a bug fix inside heap_update() with no new syntax, GUC, catalog object, dump format or psql change, so the user-facing contract is untouched. Its intended visible effect is that a DELETE of a referenced row now waits or raises the foreign-key error, which is the documented behaviour. Rui's report in message #4 says v1 makes RETURNING xmax and pgrowlocks show the updated row locked by its own transaction. That is a visible change, but it is a correctness defect for the design and safety angles, and 0001 removes it.
- replication-ondisk: 1 objection(s). The fix adds no WAL record type, catalog change or wire change. The concern is only the tuple header and multixact state that v1 writes, which persists and replicates. The page-extension path and back-branch scope could not be checked from the material.
- security: 0 objection(s). The proposal adds no privilege, role, GUC or exposure: it reorders xmax computation inside heap_update(), and the new injection point exists only in builds with injection points enabled. The lost KEY SHARE lock is a data-integrity bug that any user with insert and delete rights can hit, but the remaining gaps (message #4) are correctness issues for other angles.
- maintenance-fit: 2 objection(s). The fix is for silent foreign-key violations, so it is desirable. Its extra cost falls only on updates that leave the buffer unlocked for TOAST or page-extension work. The objections above concern long-run fragility and ownership. They are not reasons to reject the fix.
- already-exists: 0 objection(s). This is a data-integrity bug fix in heap_update(), where a KEY SHARE lock from a foreign-key check is lost during TOAST work. No existing feature, setting, documented idiom or extension prevents the orphaned child row, so from this angle the need is not already met.
- demand: 0 objection(s). No demand objection applies. This is a data-integrity bug fix: a foreign-key constraint can be violated with ordinary DML, and the report reproduces it in 12 of 12 timed trials without special setup (message #1). Every user of foreign keys on tables with TOASTed columns is exposed, and the fix adds no new user-facing feature or setting.

### Precedents retrieved

- P1: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P2: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [disallow HEAP_XMAX_COMMITTED and HEAP_XMAX_IS_LOCKED_ONLY](https://commitfest.postgresql.org/patch/3434/) (cf, Withdrawn, 2023-02-25) - judged not relevant; reason summarised from its thread
- P4: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P5: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant; reason summarised from its thread
- P6: [Forcing new tuples of updates off the page](https://commitfest.postgresql.org/patch/4440/) (cf, Withdrawn, 2024-03-15) - judged not relevant; reason summarised from its thread
- P7: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P8: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P9: [partial heap only tuples](https://commitfest.postgresql.org/patch/2981/) (cf, Returned with feedback, 2021-11-04) - judged not relevant; reason summarised from its thread
- P10: [Revert changes to CREATE TRIGGER and ALTER TABLE ADD FOREIGN KEY locking, per request from Tom.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50ce8ab9fcd49dfcbff9e19c5aa53efe0fc6dc6e) (revert, Reverted, 2005-03-24) - judged not relevant
- P11: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P12: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant; reason summarised from its thread
- P13: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P14: [BufferAlloc: don't take two simultaneous locks](https://commitfest.postgresql.org/patch/3483/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [Remove ctid from self-join examples in UPDATE and DELETE docs](https://commitfest.postgresql.org/patch/6246/) (cf, Withdrawn, 2026-01-16) - judged not relevant
- P16: [Refactor how we form HeapTuples for CatalogTuple(Insert|Update)](https://commitfest.postgresql.org/patch/6221/) (cf, Withdrawn, 2026-01-30) - judged not relevant
- P17: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P18: [Parallel tests publication and subscription might fail due to concurrent tuple update](https://commitfest.postgresql.org/patch/5459/) (cf, Rejected, 2025-03-27) - judged not relevant
- P19: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P20: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P21: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P22: [Speed up releasing of locks](https://commitfest.postgresql.org/patch/3501/) (cf, Withdrawn, 2024-05-08) - judged not relevant
- P23: [Revert lowering of lock level for ATTACH PARTITION](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8ca984b2c739e096c08f260f477792495f4dfe4) (revert, Reverted, 2018-04-12) - judged not relevant
- P24: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 94 s; cost reported by the CLI: $1.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 60 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:17:15.893Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
