# Design pre-review: Duplicate primary keys under SSI

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7197](https://commitfest.postgresql.org/patch/7197/); [thread](https://www.postgresql.org/message-id/flat/CA%2BCOZaBOiiRPmEfX00oE%3DN6HBSZVe0Y-y-ZqqaXq8BAAj1gu%2BQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). The bug is real and reproducible (messages #1 and #2), and no surviving objection is against fixing it. The open points are fixable design questions about v3: replication-ondisk-1 (summarized deleter), standards-compat-1 (false positives on back branches), replication-ondisk-2 (reset of the global variable) and standards-compat-2 (exclusion constraints). v3 has no review, and its commit message and files are not shown, so these need a committer who knows predicate.c.

Not checked: The v3 commit message, touched files and code are not shown, so its design is known only from the brief. Whether it already covers the cases in message #3 could not be checked.

## Prior attempts

None found before 2026-07-28.

## Objections a reviewer is likely to raise

### 1. Conflict check loses the deleter once it is summarized

A busy serializable workload forces SummarizeOldestCommittedSxact to move the committed deleter into the old-committed summary. Session 1 then reuses the freed key and commits with both rows visible. Nobody notices, because the anomaly appears only under load and leaves no error.

Message #3 showed that v1 looks up the deleter's sxact, gets NULL after summarization, and returns without failing. Summarization keeps only coarse conflict data, so a check for an rw-conflict to that exact transaction cannot work once the deleter is summarized. The fix then depends on how much other serializable traffic runs at the same time. Messages #4 to #7 describe v3 without saying whether it relies on the deleter's sxact or how it acts after summarization.

**Question:** When the deleting transaction has already been summarized, does v3 fail conservatively, and which isolation test forces summarization to prove it?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#4](https://www.postgresql.org/message-id/CAJ7c6TMSLKtuG8Wc7LJGo_TCCt3_6YoF8YTVaPURwOgVO3eY-A%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Back-branch fix adds new serialization failures with unproven false-positive scope

A minor release ships v3 to a stable branch. A serializable application that drops and re-creates a catalog object, or reuses a key freed by an unrelated transaction, starts getting SQLSTATE 40001 on statements that committed before. If its retry loop does not cover DDL, the deploy job fails after a routine minor upgrade.

v3 is presented as backpatchable because it keeps the ABI and "existing behaviour" (message #6), but it adds new errors by design. Message #3 showed v1 rejecting valid CREATE TABLE after a concurrent DROP, and message #5 showed an earlier PoC breaking partition-key-update-3.spec. No reply says whether v3 avoids the DDL case or limits the recheck to relations that need predicate locking. Back branches tolerate new errors only for real anomalies.

**Question:** Which existing isolation specs and Green's DDL history pass unchanged under v3, and which back branches is it meant for?

<sub>fixable, level 2, backpatchability; messages [#3](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#5](https://www.postgresql.org/message-id/CAJ7c6TMJ1ChMgfyY%2B_pT4ncAC9tf7TO1c3wuErhg-aUHqF74Pg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Per-backend global in v3 has no stated reset on error

A unique check sets the per-backend variable, then an ERROR or a savepoint rollback interrupts the insert before the variable is cleared. The next insert in that backend reads the stale value. It either raises a serialization failure that should not happen or skips one it needs, and the user sees no cause.

Message #6 says v3 adds a per-backend global variable so it can avoid an ABI change, and plans to refactor it later on master. The thread does not say where the variable is set, where it is cleared, or whether transaction abort and subtransaction abort reset it. A back-branch fix built on global state needs that pairing written down, because back branches are hard to change after release.

**Question:** Where is the v3 global variable reset on transaction abort, subtransaction abort and ERROR inside the unique check, and which test covers a savepoint rollback?

<sub>fixable, level 2, design; messages [#6](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com); `replication-ondisk-2`</sub>

### 4. Unique and exclusion constraints would give different serializable guarantees

A user replaces a primary key with EXCLUDE USING btree (k WITH =), or uses a range exclusion constraint for bookings. The same delete-then-reuse history that now fails under a unique index still commits a duplicate under the exclusion constraint. The user finds out only when the data shows two conflicting rows.

Message #3 reproduced the anomaly with an exclusion constraint, which runs its own dirty-snapshot scan outside _bt_check_unique. The v1 commit message states a root cause that covers both paths, but the fix touches only nbtree. Nobody in the thread has said whether exclusion constraints are in scope or a documented follow-up. Fixing one path and backpatching it would leave SERIALIZABLE behaviour depending on the constraint type, with no note to users.

**Question:** Does v3 cover the exclusion-constraint path, and if not, where is the remaining gap recorded for users?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com); `standards-compat-2`</sub>

### 5. No SSI-qualified reviewer has looked at the version proposed for commit

A committer backpatches v3 to all supported branches based only on its own tests. Message #3 lists cases that v3 might still miss: deferred constraints, a summarized deleter, DDL false positives and exclusion constraints. A user then hits one of them after a minor release, with no SSI expert engaged to fix it.

Bryan Green reviewed v1 only (message #3). Messages #5 to #7 change the design twice, and nobody has replied to any of them. SSI is an area few committers know well, and a backpatched change there ties every supported branch to this design.

**Question:** Which of the cases from message #3 does v3 cover, and has a committer familiar with predicate.c agreed to review it for backpatching?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/55cd61b8-ab97-4700-964f-686560a0f408%40gmail.com), [#5](https://www.postgresql.org/message-id/CAJ7c6TMJ1ChMgfyY%2B_pT4ncAC9tf7TO1c3wuErhg-aUHqF74Pg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAJ7c6TPuaFHoi9V4FByPybT-EoD0kq5eeyWmmachRh57-erRRg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJ7c6TO7zLiBezC5%2BobE9hrg4yh-GDXq79yD-nMGH0vApN_TvQ%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. When the deleting transaction has already been summarized, does v3 still raise a serialization failure, and which isolation test forces summarization?
2. Does v3 skip the check on relations that do not need predicate locking, so Green's DROP TABLE / CREATE TABLE history from message #3 still commits?
3. Does v3 catch the deferred-constraint history from message #3, where the recheck runs as UNIQUE_CHECK_EXISTING?
4. Where is the v3 per-backend global variable reset on ERROR, transaction abort and subtransaction abort?
5. Are exclusion constraints in scope for this patch or a follow-up, and which back branches does v3 target?

## Touch points

- **ABI / extension API:** v1 adds a table AM callback in tableam.h; v3 avoids it with a per-backend global variable.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Back-branch fix adds new serialization failures with unproven false-positive scope
- **backpatchability:** v3 is meant for back branches and adds new serialization failures there.
- **maintenance burden ("who maintains this in 10 years"):** The fix couples nbtree unique checks to predicate.c, and master may later differ from the back branches.
- **performance on the hot path:** The change adds a check in the nbtree unique-check path in nbtinsert.c; nobody has measured it.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows SSI (predicate.c) and nbtree unique checks; none has spoken in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost medium. The fix restores the SERIALIZABLE guarantee for users of that isolation level, and the window is narrow. v3 is unreviewed and no SSI-qualified committer is engaged; review cost is a guess because the v3 patch was not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
