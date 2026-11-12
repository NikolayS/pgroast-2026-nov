# Design pre-review: Duplicate primary keys under SSI

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7197](https://commitfest.postgresql.org/patch/7197/) (Needs review); authors: Aleksander Alekseev (a.alekseev), Andrey Borodin (x4m)
- Thread: [serializable anomaly - duplicate primary keys](https://www.postgresql.org/message-id/flat/CA%2BCOZaBOiiRPmEfX00oE%3DN6HBSZVe0Y-y-ZqqaXq8BAAj1gu%2BQ%40mail.gmail.com), started 2026-07-28
- Messages read: 7 of 7
- Patch prose read: v1-0001-Detect-SSI-conflicts-when-unique-checks-reuse-keys.patch (commit message, file list and doc hunks only; no code review)
- Subsystem invariants checked: invariants/nbtree.md

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: low)

The bug is real and reproducible (messages #1 and #2), and no surviving objection is against fixing it. The open points are fixable design questions about v3: replication-ondisk-1 (summarized deleter), standards-compat-1 (false positives on back branches), replication-ondisk-2 (reset of the global variable) and standards-compat-2 (exclusion constraints). v3 has no review, and its commit message and files are not shown, so these need a committer who knows predicate.c.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v3 commit message, touched files and code are not shown, so its design is known only from the brief. Whether it already covers the cases in message #3 could not be checked.

## The proposal, restated

**Problem.** A serializable transaction can read a row, insert a new row with the same primary key because another transaction deleted the old one, see both rows, and commit. Unique checks use SnapshotDirty, which does not take part in SSI. The bug fires only when the deleter used a secondary index, and it also affects INSERT ... ON CONFLICT DO NOTHING.

**Proposed design.** v1 has the table AM report the deleting XID when SnapshotDirty skips a tuple still visible to the transaction snapshot. SSI then fails only if an rw-conflict to that transaction already exists, and flags the transaction for abort so a savepoint rollback cannot hide the error. v3, by another author, avoids an ABI change: it checks the key against the transaction's own snapshot and uses a per-backend global variable.

**User-visible surface.**
- New serialization failures on inserts and ON CONFLICT that reuse a key freed by a concurrent serializable delete.
- v1 adds a new table AM callback in tableam.h, which out-of-tree table AMs must provide.
- New isolation test read-write-unique-5.

**What the author claims.**
- The window is narrow but plausible in delete-and-recreate or upsert workflows with long transactions.
- The bug does not corrupt the index or leave a lasting duplicate, but it breaks the SERIALIZABLE guarantee.
- v1 avoids a false positive when the rw-conflict is to an unrelated transaction.
- v3 keeps the ABI and existing behaviour, so it can be backpatched.
- An earlier Alekseev PoC caused a false positive that broke partition-key-update-3.spec; v2 and later claim to avoid it.

**Thread state.** Borodin posted v1, and Bryan Green reported several holes in it; Alekseev then took over with an ABI-preserving approach, ending with v3 and a commit message. Nobody has reviewed v3, and nobody has said whether it covers Green's cases.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Conflict check loses the deleter once it is summarized

fixable · level 2, rubric item 7 · design · `replication-ondisk-1`

A busy serializable workload forces SummarizeOldestCommittedSxact to move the committed deleter into the old-committed summary. Session 1 then reuses the freed key and commits with both rows visible. Nobody notices, because the anomaly appears only under load and leaves no error.

Message #3 showed that v1 looks up the deleter's sxact, gets NULL after summarization, and returns without failing. Summarization keeps only coarse conflict data, so a check for an rw-conflict to that exact transaction cannot work once the deleter is summarized. The fix then depends on how much other serializable traffic runs at the same time. Messages #4 to #7 describe v3 without saying whether it relies on the deleter's sxact or how it acts after summarization.

Thread: [#3 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#4 (Aleksander Alekseev, 2026-10-01)](https://www.postgresql.org/message-id/CAJ7c6TMSLKtuG8Wc7LJGo_TCCt3_6YoF8YTVaPURwOgVO3eY-A%40mail.gmail.com), [#6 (Aleksander Alekseev, 2026-10-12)](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7 (Aleksander Alekseev, 2026-10-14)](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com)

**Question:** When the deleting transaction has already been summarized, does v3 fail conservatively, and which isolation test forces summarization to prove it?

Counter-check: held. v3 asks the transaction's own snapshot, so it may not need the deleter's sxact at all. Messages #4 to #7 do not say so, and Green's summarization case has no reply and no stated test.

### 2. Back-branch fix adds new serialization failures with unproven false-positive scope

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A minor release ships v3 to a stable branch. A serializable application that drops and re-creates a catalog object, or reuses a key freed by an unrelated transaction, starts getting SQLSTATE 40001 on statements that committed before. If its retry loop does not cover DDL, the deploy job fails after a routine minor upgrade.

v3 is presented as backpatchable because it keeps the ABI and "existing behaviour" (message #6), but it adds new errors by design. Message #3 showed v1 rejecting valid CREATE TABLE after a concurrent DROP, and message #5 showed an earlier PoC breaking partition-key-update-3.spec. No reply says whether v3 avoids the DDL case or limits the recheck to relations that need predicate locking. Back branches tolerate new errors only for real anomalies.

Thread: [#3 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#5 (Aleksander Alekseev, 2026-10-08)](https://www.postgresql.org/message-id/CAJ7c6TMJ1ChMgfyY%2B_pT4ncAC9tf7TO1c3wuErhg-aUHqF74Pg%40mail.gmail.com), [#6 (Aleksander Alekseev, 2026-10-12)](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7 (Aleksander Alekseev, 2026-10-14)](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com)

**Question:** Which existing isolation specs and Green's DDL history pass unchanged under v3, and which back branches is it meant for?

Counter-check: held. Message #6 claims v2 keeps existing behaviour, which plausibly covers partition-key-update-3.spec. No message says v3 gates the check on relations that need predicate locking, so Green's DROP TABLE / CREATE TABLE history stays open for a back-branch fix.

### 3. Per-backend global in v3 has no stated reset on error

fixable · level 2, rubric item 14 · design, backpatchability · `replication-ondisk-2`

A unique check sets the per-backend variable, then an ERROR or a savepoint rollback interrupts the insert before the variable is cleared. The next insert in that backend reads the stale value. It either raises a serialization failure that should not happen or skips one it needs, and the user sees no cause.

Message #6 says v3 adds a per-backend global variable so it can avoid an ABI change, and plans to refactor it later on master. The thread does not say where the variable is set, where it is cleared, or whether transaction abort and subtransaction abort reset it. A back-branch fix built on global state needs that pairing written down, because back branches are hard to change after release.

Thread: [#6 (Aleksander Alekseev, 2026-10-12)](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7 (Aleksander Alekseev, 2026-10-14)](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com)

**Question:** Where is the v3 global variable reset on transaction abort, subtransaction abort and ERROR inside the unique check, and which test covers a savepoint rollback?

Counter-check: held. Message #6 confirms the per-backend global variable and gives no set or reset points. The stale-value scenario cannot be ruled out without the code, and v1 already had to treat savepoint rollback specially.

### 4. Unique and exclusion constraints would give different serializable guarantees

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user replaces a primary key with EXCLUDE USING btree (k WITH =), or uses a range exclusion constraint for bookings. The same delete-then-reuse history that now fails under a unique index still commits a duplicate under the exclusion constraint. The user finds out only when the data shows two conflicting rows.

Message #3 reproduced the anomaly with an exclusion constraint, which runs its own dirty-snapshot scan outside _bt_check_unique. The v1 commit message states a root cause that covers both paths, but the fix touches only nbtree. Nobody in the thread has said whether exclusion constraints are in scope or a documented follow-up. Fixing one path and backpatching it would leave SERIALIZABLE behaviour depending on the constraint type, with no note to users.

Thread: [#3 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com)

**Question:** Does v3 cover the exclusion-constraint path, and if not, where is the remaining gap recorded for users?

Counter-check: held. Message #3 says the gap predates the patch and that fcff8a57519 was also nbtree-only, so a follow-up is acceptable. The objection asks only for a scope statement, and no later message gives one.

### 5. No SSI-qualified reviewer has looked at the version proposed for commit

fixable · level 2, rubric item 16 · maintenance burden ("who maintains this in 10 years"), backpatchability · `maintenance-fit-3`

A committer backpatches v3 to all supported branches based only on its own tests. Message #3 lists cases that v3 might still miss: deferred constraints, a summarized deleter, DDL false positives and exclusion constraints. A user then hits one of them after a minor release, with no SSI expert engaged to fix it.

Bryan Green reviewed v1 only (message #3). Messages #5 to #7 change the design twice, and nobody has replied to any of them. SSI is an area few committers know well, and a backpatched change there ties every supported branch to this design.

Thread: [#3 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#5 (Aleksander Alekseev, 2026-10-08)](https://www.postgresql.org/message-id/CAJ7c6TMJ1ChMgfyY%2B_pT4ncAC9tf7TO1c3wuErhg-aUHqF74Pg%40mail.gmail.com), [#6 (Aleksander Alekseev, 2026-10-12)](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7 (Aleksander Alekseev, 2026-10-14)](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com)

**Question:** Which of the cases from message #3 does v3 cover, and has a committer familiar with predicate.c agreed to review it for backpatching?

Counter-check: held. The CommitFest entry lists one reviewer, who saw only v1, and messages #5 to #7 have no replies. This belongs in the committability field and must not raise the verdict on the idea.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | v1 adds a table AM callback in tableam.h; v3 avoids it with a per-backend global variable. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Back-branch fix adds new serialization failures with unproven false-positive scope |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | v3 is meant for back branches and adds new serialization failures there. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The fix couples nbtree unique checks to predicate.c, and master may later differ from the back branches. |
| performance on the hot path | **yes** | The change adds a check in the nbtree unique-check path in nbtinsert.c; nobody has measured it. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. When the deleting transaction has already been summarized, does v3 still raise a serialization failure, and which isolation test forces summarization?
2. Does v3 skip the check on relations that do not need predicate locking, so Green's DROP TABLE / CREATE TABLE history from message #3 still commits?
3. Does v3 catch the deferred-constraint history from message #3, where the recheck runs as UNIQUE_CHECK_EXISTING?
4. Where is the v3 per-backend global variable reset on ERROR, transaction abort and subtransaction abort?
5. Are exclusion constraints in scope for this patch or a follow-up, and which back branches does v3 target?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows SSI (predicate.c) and nbtree unique checks; none has spoken in the visible thread.
- **Liability:** The owner signs for new serialization failures in every back branch that gets the fix, including any false positives, and for SSI code that differs between master and back branches if master is refactored later.
- **Author follow-up:** Alekseev took over from v1 and posted through v3 with a commit message, and the CommitFest entry lists both authors; no reply to Green's cases is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The fix restores the SERIALIZABLE guarantee for users of that isolation level, and the window is narrow. v3 is unreviewed and no SSI-qualified committer is engaged; review cost is a guess because the v3 patch was not read.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-branch fix relies on a per-backend global with no stated lifecycle. Refuted (not specific): Duplicate of replication-ondisk-2. Its one added point, different code on master and back branches, is common practice for ABI-preserving back-branch fixes and has no failure scenario of its own.
- `maintenance-fit-2` No statement that the new check costs nothing outside serializable. Refuted (no failure scenario): The scenario depends on v3 lacking a serializable gate, which the proposal does not suggest. Even ungated, the check runs only when a unique insert finds an equal-key tuple that SnapshotDirty skips, a path that already does a heap fetch.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier rejected, returned or reverted attempt at this idea. Messages #1 and #2 point only to fcff8a57519 and a related ON CONFLICT thread as related work, and neither is reported as a rejection.
- standards-compat: 2 objection(s). The proposal adds no syntax, GUC, catalog object or pg_dump state. New serialization failures for real anomalies follow the SERIALIZABLE contract and are not a compatibility objection in themselves.
- replication-ondisk: 2 objection(s). There is no on-disk format, WAL, catalog, wire protocol or pg_upgrade change, and v3 drops the v1 table AM callback. The nbtree VACUUM bookkeeping invariant does not apply here.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It adds serialization failures only where a transaction's own snapshot still sees the key, so it reveals no more than unique checks and SSI errors already do. Bryan Green reviewed v1 for security in message #3 and raised only non-security points, and no one has reviewed v3 the same way.
- maintenance-fit: 3 objection(s). The goal is a real SERIALIZABLE correctness bug and belongs in core. These objections concern the shape of v3 and who will own it, not whether to fix the bug.
- already-exists: 0 objection(s). This is a correctness fix for SERIALIZABLE, and no existing feature, setting or extension restores the guarantee for unique checks that rely on a committed delete. Explicit row locks in application code avoid only some cases and do not fix the isolation level, so this angle has nothing to raise.
- demand: 0 objection(s). This is a correctness fix for a documented guarantee, so lack of user demand does not apply. Message #1 gives a reproducible SERIALIZABLE anomaly, and Message #2 confirms it, including the ON CONFLICT DO NOTHING case. Users who choose SERIALIZABLE rely on that guarantee without asking for it, and the new serialization failures land only on serializable transactions that hit the anomaly. Whether v3 covers the cases in Message #3 is a design question for other angles.

### Precedents retrieved

- P1: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P2: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1419/) (cf, Returned with feedback, 2018-03-06) - judged not relevant; reason summarised from its thread
- P3: [WIP: Pgbench Errors and serialization/deadlock retries](https://commitfest.postgresql.org/patch/1645/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P4: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant; reason summarised from its thread
- P5: [Statement-level Triggers For Uniqueness Checks](https://commitfest.postgresql.org/patch/1946/) (cf, Withdrawn, 2019-01-29) - judged not relevant; reason summarised from its thread
- P6: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P7: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P8: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/1745/) (cf, Returned with feedback, 2019-06-28) - judged not relevant; reason summarised from its thread
- P9: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/2186/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P10: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P11: [Add callback table access method to reset filenode when dropping relation](https://commitfest.postgresql.org/patch/3073/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P12: [Unique Joins](https://commitfest.postgresql.org/patch/129/) (cf, Returned with feedback, 2016-04-09) - judged not relevant
- P13: [Remove duplicated smgrclose call](https://commitfest.postgresql.org/patch/5149/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P14: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant
- P15: [Revert workarounds for -Wmissing-braces false positives on old GCC](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=618c64ffd3967cb5313b4b11e1e1043a074f2139) (revert, Reverted, 2025-03-20) - judged not relevant
- P16: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P17: [Predicate locking in Gin index](https://commitfest.postgresql.org/patch/1199/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P18: [Add checkpoint stats of snapshot and mapping files of pg_logical dir](https://commitfest.postgresql.org/patch/3389/) (cf, Withdrawn, 2022-10-11) - judged not relevant
- P19: [Predicate locking in gin index](https://commitfest.postgresql.org/patch/1581/) (cf, Rejected, 2018-03-02) - judged not relevant
- P20: [Change tuple table slot for Unique node to "virtual"](https://commitfest.postgresql.org/patch/4531/) (cf, Withdrawn, 2024-03-28) - judged not relevant
- P21: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P22: [skip replication slot snapshot/map file removal during end-of-recovery checkpoint](https://commitfest.postgresql.org/patch/3491/) (cf, Withdrawn, 2022-01-10) - judged not relevant
- P23: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P24: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 131 s; cost reported by the CLI: $1.40
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 81 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:25.069Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
