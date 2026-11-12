# Design pre-review: pg_stat_xact_user_tables stat not currect during the transaction

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6619](https://commitfest.postgresql.org/patch/6619/) (Needs review); authors: Xuneng Zhou (dylanzhou)
- Thread: [BUG #19439: pg_stat_xact_user_tables stat not currect during the transaction](https://www.postgresql.org/message-id/flat/19439-8f02018f9fc3a240%40postgresql.org), started 2026-03-25
- Messages read: 8 of 8
- Patch prose read: v1-0001-Fix-pg_stat_xact_-views-leaking-stats-across-tran.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal fits the view names and a fair reading of the documentation, but neither design is ready for code review. maintenance-fit-1 is decisive: v2 costs 44% TPS at 5000 tables for backends that never read the views, and v1 adds an unmeasured branch to the counter macros. The author has not chosen between them. standards-compat-1 and standards-compat-2 show that the reporting contract and the target branches are also undecided, and demand-1 asks whether a documentation change would be enough.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the v1 overhead in the counter macros, the layout of the baseline fields and the handling of subtransactions could not be checked. Only 3 of the 8 messages were shown in full; the v2 details come from the brief and the objections.

## The proposal, restated

**Problem.** The pg_stat_xact_* views show counters that carry over from earlier transactions of the same backend. The case is deterministic when several top-level transactions run in one simple-query message, because no ReadyForQuery boundary lets pending stats flush. The reporter and the author read the documentation as promising per-transaction numbers.

**Proposed design.** Each backend-local pending stats entry keeps a transaction baseline, and the view accessors subtract it so they report only the current transaction's delta. v1 takes the baseline lazily at each counter-increment site, keyed by MyProc->vxid.lxid. v2 instead sweeps all pending entries at commit, abort and PREPARE TRANSACTION.

**User-visible surface.**
- pg_stat_xact_all_tables, pg_stat_xact_sys_tables and pg_stat_xact_user_tables report only the current transaction's counts.
- pg_stat_xact_user_functions reports only the current transaction's function calls.
- C API: the patch adds the PgStat_FunctionPending struct, which wraps PgStat_FunctionCounts in pending function stats entries.

**What the author claims.**
- The author reproduced the behaviour on HEAD and on v13, so it predates the stats rewrite in commit 5891c7a8.
- With v1, the hot-path check costs a single integer comparison after the first touch per entry per transaction.
- v2 costs grow linearly with the number of pending entries per backend, about 0.03-0.04 us per entry per transaction in a cassert build.
- v2 loses 44% TPS with 5000 tables and 1 row per table, 7% with 1000 tables, and about nothing with 100 tables.
- Regular pg_regress files cannot show the bug, so the test is a psql TAP test that uses psql -c.

**Thread state.** Only the author and the reporter took part. A reviewer signed up in the CommitFest and later removed himself, the patch needed a rebase in May, and the author has not chosen between the lazy-baseline design and the eager design.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-03-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Both designs charge every backend for a rarely used view

fixable · level 2, rubric item 4 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A pooled application keeps long-lived sessions that touch thousands of relations and then run short transactions. It never queries pg_stat_xact_*. With v2, its throughput drops by up to 44% after an upgrade, and the operator sees this only as higher commit latency.

Message #8 shows the v2 sweep costs O(pending entries) at every top-level commit, abort and PREPARE, with -44% TPS at 5000 tables and -7% at 1000. v1 moves the cost into the counter macros on scan and buffer-read paths, and message #5 names this as a drawback. In both designs every backend pays, and only sessions that query the xact views gain. The v1 overhead is not measured on an optimized build.

Thread: [#5 (Xuneng Zhou, 2026-03-30)](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8 (Xuneng Zhou, 2026-05-13)](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com)

**Question:** Can the baseline cost fall only on sessions that query pg_stat_xact_* views, with no measurable cost on an optimized build for backends that never query them?

Counter-check: held. Message #8 measures the v2 cost on backends that never read the views, and message #5 names the v1 hot-path branch as a drawback. The thread shows no v1 measurement on an optimized build, and the author has not chosen a design.

### 2. Changed view semantics come with no documentation of the new contract

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-1`

A monitoring script reads pg_stat_xact_user_tables after a rollback to savepoint, or after a function that ran its own subtransactions. The numbers it gets depend on where the baseline was taken. The documentation does not say what is reported, so the user cannot tell a correct count from a new bug.

Message #2 calls the current documentation "somewhat ambiguous", and the fix resolves that ambiguity only in code. The touched-file list for v1 has no doc hunk, and no v2 commit message or doc change is shown. The patch redefines what four user-visible views mean, so the monitoring-stats page needs to state the scope (top-level transaction, subtransactions, PREPARE TRANSACTION). Then reviewers and users can check behaviour against it.

Thread: [#2 (Xuneng Zhou, 2026-03-27)](https://www.postgresql.org/message-id/CABPTF7Wx%3DWTHa67PJUnH8VaJf-svrcFjt1ZgOVrqWzfgwDuMCA%40mail.gmail.com), [#3 (Xuneng Zhou, 2026-03-28)](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com), [#5 (Xuneng Zhou, 2026-03-30)](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8 (Xuneng Zhou, 2026-05-13)](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com)

**Question:** What exact wording would the monitoring-stats documentation use for the pg_stat_xact_* views, including subtransactions and PREPARE TRANSACTION?

Counter-check: held. The v1 file list has no documentation hunk, and message #2 itself calls the current wording ambiguous. The thread does not define the scope for subtransactions or PREPARE TRANSACTION, so the request for wording stands.

### 3. Bugfix tag implies back-patch, but the change alters stable-branch structs and costs

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API, performance on the hot path · `standards-compat-2`

The fix is back-patched as a bug fix. An extension built against a minor release that reads pending function stats through pgstat.h now sees PgStat_FunctionPending in place of PgStat_FunctionCounts. A minor update also adds per-commit work, and v2 costs 44% TPS in one measured workload. Users find out after upgrading a minor release.

The CommitFest entry is tagged Bugfix, and message #4 shows the behaviour goes back to v13. That invites back-patching through every supported branch. The thread never says whether this is master-only. v1 changes the pending-entry layout and adds a branch at every counter-increment macro in pgstat.h. v2 adds a sweep at every commit, with the regression shown in message #8. A committer would likely treat either change as master-only and want the decision stated.

Thread: [#4 (Xuneng Zhou, 2026-03-29)](https://www.postgresql.org/message-id/CABPTF7VSNTvHTpS4ga1MfLYO-%3Dxt7mLQ0ONEFobRERnhzXSzxw%40mail.gmail.com), [#5 (Xuneng Zhou, 2026-03-30)](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8 (Xuneng Zhou, 2026-05-13)](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com)

**Question:** Is this proposed for master only, and if back-patched, how does it avoid changing pgstat.h struct layouts and commit-path cost in stable branches?

Counter-check: held. The entry is tagged Bugfix, message #4 shows the behaviour on v13, and the thread does not state a target branch. The extension scenario is weak because the wrapper keeps PgStat_FunctionCounts at offset 0, but the stable-branch cost concern stands.

### 4. Every future counter site must repeat the lazy-baseline call

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A later patch adds a counter to PgStat_TableCounts and increments it without calling pgstat_ensure_xact_baseline(). No test fails. Users then see that one column carry over between transactions in pg_stat_xact_* views, and the report arrives as a bug years later.

Message #3 places the helper in 7 macros and 4 functions in pgstat_relation.c. Message #5 states that a missed call makes the counter leak silently. Message #8 then leans back toward the lazy-baseline design but does not add a safeguard. The TAP test covers only n_tup_ins, seq_scan and function calls, so it would not catch a new counter.

Thread: [#3 (Xuneng Zhou, 2026-03-28)](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com), [#5 (Xuneng Zhou, 2026-03-30)](https://www.postgresql.org/message-id/CABPTF7XO101mBvA9u01nHkvt1gaQ3i60hypYOvNp6DdOFnY0jQ%40mail.gmail.com), [#8 (Xuneng Zhou, 2026-05-13)](https://www.postgresql.org/message-id/CABPTF7WUzwywrDmB%2BW%3DSm%3DynzQkcS83p2z-Ywt8W8XdKhXOPcQ%40mail.gmail.com)

**Question:** What makes a new counter added without a baseline call fail loudly in a test or an assertion?

Counter-check: held. Message #5 states this risk in the author's own words, and the TAP test covers only three counters. It applies to the lazy-baseline design only, but message #8 leans back toward that design without adding a safeguard.

### 5. One report and no other users; a documentation fix may be enough

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adopts a per-entry baseline in the stats code to serve one reporter's psql -c test. Every later change to PgStat_TableCounts must keep the baseline right. The users who notice the difference are those who run several top-level transactions in one message and read pg_stat_xact_* between them.

The only demand in the thread is the bug report in message #1, a hand-written test with no production use behind it. Message #2 calls the documentation "somewhat ambiguous", so rewording it to describe the current behaviour costs nothing, and users who need exact deltas can read the view at the start and end of the transaction and subtract. Message #3 says the rate limiter can keep entries pending across ReadyForQuery, so the affected group may be larger, but no one in the thread shows such a case.

Thread: [#1 (PG Bug reporting form, 2026-03-25)](https://www.postgresql.org/message-id/19439-8f02018f9fc3a240%40postgresql.org), [#2 (Xuneng Zhou, 2026-03-27)](https://www.postgresql.org/message-id/CABPTF7Wx%3DWTHa67PJUnH8VaJf-svrcFjt1ZgOVrqWzfgwDuMCA%40mail.gmail.com), [#3 (Xuneng Zhou, 2026-03-28)](https://www.postgresql.org/message-id/CABPTF7Wb6VOac1EFopGzhAtiHOU8_XRkFvhC6BnSj%3DjACBB%2BGw%40mail.gmail.com)

**Question:** Which monitoring tools or applications read pg_stat_xact_* after an earlier transaction in the same session and act on the numbers, and why does a documentation change not serve them?

Counter-check: held. The thread has one reporter and no production case, and reading the view twice and subtracting does the job. The flush rate limiter may make carry-over common in ordinary sessions, but the thread does not show this, so asking for evidence is fair.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds PgStat_FunctionPending and changes the counter macros in pgstat.h, which extensions compile against. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Changed view semantics come with no documentation of the new contract |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix and the behaviour goes back to v13, but the thread names no target branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | With v1, every future counter-increment site must call pgstat_ensure_xact_baseline(), and nothing fails if one is missed. |
| performance on the hot path | **yes** | v1 adds a check to the counter-increment macros; v2 sweeps all pending entries at every commit, abort and PREPARE TRANSACTION. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | A rewording of the monitoring-stats page to describe the current behaviour is an open alternative. |

## 5. Questions the author must answer before anyone reviews code

1. Which design do you propose for commit, the lazy-baseline design of v1 or the commit-time sweep of v2?
2. Can the baseline work run only in sessions that read a pg_stat_xact_* view, so that other backends pay nothing on an optimized build?
3. What wording would the monitoring-stats page use for the pg_stat_xact_* views, including subtransactions, rollback to savepoint and PREPARE TRANSACTION?
4. Is this change for master only, and if not, how do stable branches avoid the pgstat.h struct changes and the added commit-path cost?
5. With the lazy-baseline design, what test or assertion fails when a new counter is incremented without a call to pgstat_ensure_xact_baseline()?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the cumulative statistics code (pgstat pending entries and their flush path) would have to own this. None has spoken in the visible thread.
- **Liability:** The owner signs for a per-entry baseline in the pgstat hot path or commit path, for any performance report from backends with many pending entries, and for the back-patch decision.
- **Author follow-up:** The author posted two versions and benchmarks within the thread. The CommitFest entry has needed a rebase since May, and the material shows no rebase after that.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** One reporter reads these views between transactions in one session, and both designs put cost on all backends, with v2 measured at 44% TPS loss at 5000 tables. The patch is small and confined to pgstat, but it touches counter macros on common paths.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Back-patching either design changes pgstat structs and macros that extensions use. Refuted (not specific): Duplicate of standards-compat-2. The memory corruption claim also has no support in the material, which does not show where the baseline fields sit in the pending entry.
- `demand-2` Every backend pays for a view that few sessions read. Refuted (not specific): Duplicate of maintenance-fit-1, which uses the same measurements from message #8 and also notes the missing v1 measurement on an optimized build.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier proposal with the same or a similar idea. The thread itself points to no earlier discussion of scoping pg_stat_xact_* views per transaction, so this angle has nothing to raise.
- standards-compat: 2 objection(s). Two fixable objections only. No new syntax, GUC, catalog object or pg_dump state is involved, and existing users who depend on the leaking counts are unlikely.
- replication-ondisk: 1 objection(s). Pending stats are backend-local memory only. The shared stats file format, WAL, the catalogs, replication and the wire protocol are unchanged, so my only concern from this angle is ABI and back-patching.
- security: 0 objection(s). The pg_stat_xact_* views read only the calling backend's own pending stats, and the patch changes only how those numbers are scoped in time. It adds no privilege, role or new exposure of other sessions' data. The v2 sweep cost falls on the backend that holds the pending entries, so it is a performance question for another angle and gives no cross-user denial of service.
- maintenance-fit: 2 objection(s). Two objections at level 2. No committer who knows pgstat has engaged in the thread, so the cleanup owner is open; this is a committability point and not an objection.
- already-exists: 0 objection(s). No existing feature, contrib module or extension gives transaction-scoped table or function counters; pg_stat_xact_* views are that mechanism, and the patch repairs them without adding a second one. A client could snapshot the view at BEGIN and subtract it, but that workaround for a bug is not an alternative a committer would offer.
- demand: 2 objection(s). Two demand objections are raised; no precedent in the material applies.

### Precedents retrieved

- P1: [Revert "Add redo LSN to pgstats files"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5721e5453ebc59360b6756fe72d7499c4a02177c) (revert, Reverted, 2025-03-16) - judged not relevant
- P2: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P3: [Flush pgstats file during checkpoints](https://commitfest.postgresql.org/patch/5047/) (cf, Returned with feedback, 2024-10-02) - judged not relevant; reason summarised from its thread
- P4: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant; reason summarised from its thread
- P5: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [pgstattuple: add test for coverage](https://commitfest.postgresql.org/patch/3793/) (cf, Returned with feedback, 2022-11-22) - judged not relevant; reason summarised from its thread
- P7: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P8: [System username in pg_stat_activity](https://commitfest.postgresql.org/patch/4769/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant; reason summarised from its thread
- P10: [Revert XactLockTableWait context setup in conditional multixact wait](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3adba736620c85112e0f9f6d043d3574e1c5830f) (revert, Reverted, 2014-08-25) - judged not relevant
- P11: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P12: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P13: [Revert "Add some temporary instrumentation to pgstat.c."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5a2bb060121f574d6587e5fc6c7f1d4e3a759351) (revert, Reverted, 2012-05-15) - judged not relevant
- P14: [An attempt to avoid locally-committed-but-not-replicated-to-standby-transactions in synchronous replication](https://commitfest.postgresql.org/patch/3665/) (cf, Returned with feedback, 2023-08-02) - judged not relevant
- P15: [Report the query string that caused a memory error under Valgrind](https://commitfest.postgresql.org/patch/4156/) (cf, Withdrawn, 2023-01-31) - judged not relevant
- P16: [Revert "Fix issues with Windows' stat() for files pending on deletion"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c9b46c090e76c62f24563b9be2c34e6b92e9329) (revert, Reverted, 2021-07-12) - judged not relevant
- P17: [Revert support for improved tracking of nested queries](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f85f6ab051b7cf6950247e5fa6072c4130613555) (revert, Reverted, 2025-06-12) - judged not relevant
- P18: [User functions for building SCRAM secrets](https://commitfest.postgresql.org/patch/3988/) (cf, Returned with feedback, 2024-01-27) - judged not relevant
- P19: [Revert exports.list change pending closer study.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=36b870623698332144c73e950d6a8f91e883d708) (revert, Reverted, 2006-12-28) - judged not relevant
- P20: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P22: [Eliminating "Permission denied" errors on stat() (on Windows)](https://commitfest.postgresql.org/patch/3058/) (cf, Withdrawn, 2021-12-03) - judged not relevant
- P23: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant
- P24: [pg_stat_walwrites system view](https://commitfest.postgresql.org/patch/1003/) (cf, Returned with feedback, 2017-11-27) - judged not relevant

### Warnings

- tone lint: 6 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 149 s; cost reported by the CLI: $1.57
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 81 text field(s) checked, 6 finding(s), 6 sentence(s) rewritten (repair call: done), 6 left
  - `brief.thread_state`: about the author ("lazy")
  - `questions[0]`: about the author ("lazy")
  - `questions[4]`: about the author ("lazy")
  - `objections[3].title`: about the author ("lazy")
  - `objections[3].argument`: about the author ("lazy")
  - `objections[3].critic.reasoning`: about the author ("lazy")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:56:38.663Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
