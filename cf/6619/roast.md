# Design pre-review: pg_stat_xact_user_tables stat not currect during the transaction

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6619](https://commitfest.postgresql.org/patch/6619/); [thread](https://www.postgresql.org/message-id/flat/19439-8f02018f9fc3a240%40postgresql.org), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal fits the view names and a fair reading of the documentation, but neither design is ready for code review. maintenance-fit-1 is decisive: v2 costs 44% TPS at 5000 tables for backends that never read the views, and v1 adds an unmeasured branch to the counter macros. The author has not chosen between them. standards-compat-1 and standards-compat-2 show that the reporting contract and the target branches are also undecided, and demand-1 asks whether a documentation change would be enough.

Not checked: The code was not read, so the v1 overhead in the counter macros, the layout of the baseline fields and the handling of subtransactions could not be checked. Only 3 of the 8 messages were shown in full; the v2 details come from the brief and the objections.

## Prior attempts

None found before 2026-03-25.

## Objections a reviewer is likely to raise

### 1. Both designs charge every backend for a rarely used view

A pooled application keeps long-lived sessions that touch thousands of relations and then run short transactions. It never queries pg_stat_xact_*. With v2, its throughput drops by up to 44% after an upgrade, and the operator sees this only as higher commit latency.

Message #8 shows the v2 sweep costs O(pending entries) at every top-level commit, abort and PREPARE, with -44% TPS at 5000 tables and -7% at 1000. v1 moves the cost into the counter macros on scan and buffer-read paths, and message #5 names this as a drawback. In both designs every backend pays, and only sessions that query the xact views gain. The v1 overhead is not measured on an optimized build.

**Question:** Can the baseline cost fall only on sessions that query pg_stat_xact_* views, with no measurable cost on an optimized build for backends that never query them?

<sub>fixable, level 2, performance on the hot path; messages [#5](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Changed view semantics come with no documentation of the new contract

A monitoring script reads pg_stat_xact_user_tables after a rollback to savepoint, or after a function that ran its own subtransactions. The numbers it gets depend on where the baseline was taken. The documentation does not say what is reported, so the user cannot tell a correct count from a new bug.

Message #2 calls the current documentation "somewhat ambiguous", and the fix resolves that ambiguity only in code. The touched-file list for v1 has no doc hunk, and no v2 commit message or doc change is shown. The patch redefines what four user-visible views mean, so the monitoring-stats page needs to state the scope (top-level transaction, subtransactions, PREPARE TRANSACTION). Then reviewers and users can check behaviour against it.

**Question:** What exact wording would the monitoring-stats documentation use for the pg_stat_xact_* views, including subtransactions and PREPARE TRANSACTION?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#2](https://www.postgresql.org/message-id/CABPTF7Wx%3DWTHa67PJUnH8VaJf-svrcFjt1ZgOVrqWzfgwDuMCA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Bugfix tag implies back-patch, but the change alters stable-branch structs and costs

The fix is back-patched as a bug fix. An extension built against a minor release that reads pending function stats through pgstat.h now sees PgStat_FunctionPending in place of PgStat_FunctionCounts. A minor update also adds per-commit work, and v2 costs 44% TPS in one measured workload. Users find out after upgrading a minor release.

The CommitFest entry is tagged Bugfix, and message #4 shows the behaviour goes back to v13. That invites back-patching through every supported branch. The thread never says whether this is master-only. v1 changes the pending-entry layout and adds a branch at every counter-increment macro in pgstat.h. v2 adds a sweep at every commit, with the regression shown in message #8. A committer would likely treat either change as master-only and want the decision stated.

**Question:** Is this proposed for master only, and if back-patched, how does it avoid changing pgstat.h struct layouts and commit-path cost in stable branches?

<sub>fixable, level 2, backpatchability; messages [#4](https://www.postgresql.org/message-id/CABPTF7VSNTvHTpS4ga1MfLYO-%3Dxt7mLQ0ONEFobRERnhzXSzxw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Every future counter site must repeat the lazy-baseline call

A later patch adds a counter to PgStat_TableCounts and increments it without calling pgstat_ensure_xact_baseline(). No test fails. Users then see that one column carry over between transactions in pg_stat_xact_* views, and the report arrives as a bug years later.

Message #3 places the helper in 7 macros and 4 functions in pgstat_relation.c. Message #5 states that a missed call makes the counter leak silently. Message #8 then leans back toward the lazy-baseline design but does not add a safeguard. The TAP test covers only n_tup_ins, seq_scan and function calls, so it would not catch a new counter.

**Question:** What makes a new counter added without a baseline call fail loudly in a test or an assertion?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 5. One report and no other users; a documentation fix may be enough

The project adopts a per-entry baseline in the stats code to serve one reporter's psql -c test. Every later change to PgStat_TableCounts must keep the baseline right. The users who notice the difference are those who run several top-level transactions in one message and read pg_stat_xact_* between them.

The only demand in the thread is the bug report in message #1, a hand-written test with no production use behind it. Message #2 calls the documentation "somewhat ambiguous", so rewording it to describe the current behaviour costs nothing, and users who need exact deltas can read the view at the start and end of the transaction and subtract. Message #3 says the rate limiter can keep entries pending across ReadyForQuery, so the affected group may be larger, but no one in the thread shows such a case.

**Question:** Which monitoring tools or applications read pg_stat_xact_* after an earlier transaction in the same session and act on the numbers, and why does a documentation change not serve them?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/19439-8f02018f9fc3a240%40postgresql.org), [#2](https://www.postgresql.org/message-id/CABPTF7Wx%3DWTHa67PJUnH8VaJf-svrcFjt1ZgOVrqWzfgwDuMCA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which design do you propose for commit, the lazy-baseline design of v1 or the commit-time sweep of v2?
2. Can the baseline work run only in sessions that read a pg_stat_xact_* view, so that other backends pay nothing on an optimized build?
3. What wording would the monitoring-stats page use for the pg_stat_xact_* views, including subtransactions, rollback to savepoint and PREPARE TRANSACTION?
4. Is this change for master only, and if not, how do stable branches avoid the pgstat.h struct changes and the added commit-path cost?
5. With the lazy-baseline design, what test or assertion fails when a new counter is incremented without a call to pgstat_ensure_xact_baseline()?

## Touch points

- **ABI / extension API:** The patch adds PgStat_FunctionPending and changes the counter macros in pgstat.h, which extensions compile against.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Changed view semantics come with no documentation of the new contract
- **backpatchability:** The entry is tagged Bugfix and the behaviour goes back to v13, but the thread names no target branch.
- **maintenance burden ("who maintains this in 10 years"):** With v1, every future counter-increment site must call pgstat_ensure_xact_baseline(), and nothing fails if one is missed.
- **performance on the hot path:** v1 adds a check to the counter-increment macros; v2 sweeps all pending entries at every commit, abort and PREPARE TRANSACTION.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** A rewording of the monitoring-stats page to describe the current behaviour is an open alternative.

## Who would commit it, and when

- Owner: silent. A committer who knows the cumulative statistics code (pgstat pending entries and their flush path) would have to own this. None has spoken in the visible thread.
- Priority: needs design rethink; audience niche; review cost medium. One reporter reads these views between transactions in one session, and both designs put cost on all backends, with v2 measured at 44% TPS loss at 5000 tables. The patch is small and confined to pgstat, but it touches counter macros on common paths.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
