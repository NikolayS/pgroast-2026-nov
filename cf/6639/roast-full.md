# Design pre-review: Track skipped vacuum and analyze in pg_stat_all_tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6639](https://commitfest.postgresql.org/patch/6639/) (Needs review); authors: Yugo Nagata (yugo.nagata)
- Thread: [Track skipped tables during autovacuum and autoanalyze](https://www.postgresql.org/message-id/flat/20260324151133.7940a5c1f2ebd594d54da481%40sraoss.co.jp), started 2026-03-24
- Messages read: 51 of 51
- Patch prose read: v20-0002-Add-injection-point-test-for-vacuum-skip_locked-.patch, v20-0001-Track-skipped-vacuum-and-analyze-activity-per-re.patch, 0001-Track-skipped-tables-during-autovacuum-and-autoanaly.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, and no surviving objection is fatal. The decisive point is security-1: message #51 shows that a role without privileges on a table can bump its skip counters, and v20 does not fix it. The other surviving points (already-exists-1, maintenance-fit-1, maintenance-fit-2) are fixable scope and layout questions that a committer who knows the cumulative statistics code can settle.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check the order of the lock attempt, the privilege check and the stats report in vacuum.c. It also could not check the syscache lookup that guards against a concurrent drop, or the real size of the stats entry.

## The proposal, restated

**Problem.** When autovacuum, autoanalyze or a manual VACUUM/ANALYZE with SKIP_LOCKED skips a table because the lock is not available, Postgres only writes a log message. Users who monitor tables that have not been vacuumed for a long time cannot see per table whether autovacuum is trying and being skipped, or not running at all.

**Proposed design.** The patch adds eight per-relation statistics fields: a last-skip timestamp and a skip counter for each of manual vacuum, autovacuum, manual analyze and autoanalyze. They are exposed in pg_stat_all_tables. The skip is reported when the conditional lock fails. A second syscache lookup drops a stats entry created for a concurrently dropped relation, and relkinds that the view cannot display are filtered out.

**User-visible surface.**
- New pg_stat_all_tables columns last_lock_skipped_vacuum, last_lock_skipped_autovacuum, last_lock_skipped_analyze, last_lock_skipped_autoanalyze.
- New pg_stat_all_tables columns lock_skipped_vacuum_count, lock_skipped_autovacuum_count, lock_skipped_analyze_count, lock_skipped_autoanalyze_count.
- Eight new pg_stat_get_* functions, for example pg_stat_get_last_lock_skipped_vacuum_time.
- A changed stats file format that needs a PGSTAT_FILE_FORMAT_ID bump, left for the committer.
- A documentation note that a skipped manual command on a parent table is recorded only on the parent, not on its children.

**What the author claims.**
- The timestamps show that autovacuum is attempting to run and that the lack of progress comes from repeated skips.
- The counters help dashboards spot spikes in skipped autovacuum and autoanalyze.
- The fields count lock skips only, so a future skip reason would get its own counter.
- All review comments up to v19 are addressed, and the author asks for Ready for Committer status.

**Thread state.** Sami Imseih co-develops the patch, and Shinya Kato and Zsolt Parragi reviewed it through v20. Michael Paquier backs the timestamps. vignesh C then reported that any role that can see the schema can bump a table's skip counters without privileges, and no one has answered.

## 2. Prior attempts

### More stats about skipped/executed vacuums (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/1351/](https://commitfest.postgresql.org/patch/1351/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20171031.184937.57727308.horiguchi.kyotaro@lab.ntt.co.jp](https://www.postgresql.org/message-id/flat/20171031.184937.57727308.horiguchi.kyotaro@lab.ntt.co.jp)
- **Date:** decided 2017-11-27; started 2017-10-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Only the author's repost of the patch is shown, with no review or reply.
- **Reason (quote):** quote unavailable
- **What is different this time:** P5 added many vacuum outcome columns, including a successive-failure count, and took a lock on each table inside the view. This proposal counts only lock skips and computes nothing at view time.

### Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes (related)

- **Link:** [https://commitfest.postgresql.org/patch/5238/](https://commitfest.postgresql.org/patch/5238/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAECtzeXXuMkw-RVGTWvHGOJsmFdsRY+jK0ndQa80sw46y2uvVQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAECtzeXXuMkw-RVGTWvHGOJsmFdsRY+jK0ndQa80sw46y2uvVQ@mail.gmail.com)
- **Date:** decided 2024-12-11; started 2024-08-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers doubted that per-table and per-index counts of parallel scans help with tuning, since parallel plans with Gather nodes are usually analyzed per statement. Michael Paquier added that the metrics committed in e7a9496de906 and 5d4298e75f25 may already cover the need. Robert Haas found no clear use case for these counters. The author agreed that the committed metrics were good enough for now.
- **Reason (quote):** "I think the question to which we don't have a clear answer is: for what purpose would you want to be able to distinguish parallel and non-parallel scans on a per-table basis?" ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ5oiVFF1%3DAqmESrJo4YPbk9WO8n%2BNnLPCm6wNrnoEooQ%40mail.gmail.com))
- **What is different this time:** P7 added per-table counters without a clear per-table use. This proposal states its use: telling a table that autovacuum keeps skipping from one it never tries.


## 3. Objections a reviewer is likely to raise

### 1. Roles without privileges on a table can bump its lock-skip statistics

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A role with only USAGE on the schema runs `vacuum (skip_locked) t` in a loop while the owner's batch job holds a lock. The lock_skipped_vacuum_count of t grows and last_lock_skipped_vacuum moves forward. A DBA alerting on these columns sees false spikes or loses the real signal, and cannot tell who caused them.

Message #51 shows the skip is recorded before vacuum_is_permitted_for_relation(), and includes a test that reproduces it. Message #42 reported the same gap and suggested the RangeVarGetRelidExtended() callback. Message #43 left it for a later cleanup that "someone" may take on, and v20 still has the gap with no reply to #51. Today a role without privileges on t can affect only its own WARNING output. With this patch it writes to shared per-table statistics that other roles read and alert on.

Thread: [#42 (Shinya Kato, 2026-09-16)](https://www.postgresql.org/message-id/CAOzEurRddqm-etFkG9T4auovYUDdB1bBasNJGg1jW09tXcOmTg%40mail.gmail.com), [#43 (Sami Imseih, 2026-09-16)](https://www.postgresql.org/message-id/CAN12%2BYLDPuW8iF%3DUqXyLzmjM9MkHbx2YwwKq%2B8G2-HcO8Kzyfg%40mail.gmail.com), [#51 (vignesh C, 2026-11-05)](https://www.postgresql.org/message-id/CALDaNm0%2BJ2dG5h3mX0AqqV%3DcO-vcZC0qXH_%2B7ypQ1TNWDVQhNQ%40mail.gmail.com)

**Question:** Will v21 check vacuum and analyze privileges before recording a skip, and does that check change the existing WARNING for roles without privileges?

Counter-check: held. Message #51 reproduces the gap on v20 with a test, and no one has answered it. Message #43 left the fix from message #42 as an optional follow-up, so the current patch does not handle it.

### 2. Every index and table stats entry pays for eight vacuum-only fields

fixable · level 2, rubric item 4 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A site with hundreds of thousands of relations upgrades. Each PgStat_StatTabEntry in shared memory grows by eight 8-byte fields, for indexes too, though an index never records a lock skip. The site pays this memory whether or not anyone reads the new columns.

Message #7 notes that PgStat_StatTabEntry is shared by indexes and tables, and that the vacuum and analyze fields apply only to tables. Michael proposes a separate index stats kind as a start (messages #7, #9), but v20 does not do that split and does not state the added size. Earlier entries proposing that split were returned ([P12], [P14]), with no reason recorded, so the split is not assured to land first. The cost falls on all relations, but the gain goes only to tables under lock contention.

Thread: [#7 (Michael Paquier, 2026-03-25)](https://www.postgresql.org/message-id/acRvIt2idOKeD7wv%40paquier.xyz), [#9 (Michael Paquier, 2026-03-26)](https://www.postgresql.org/message-id/acSMZ-0K_YUrmzQG%40paquier.xyz)

**Question:** How much does each relation stats entry grow, and should the index and table stats split land before this patch adds eight table-only fields?

Counter-check: held. Message #7 confirms that indexes share PgStat_StatTabEntry, so each index entry also grows by eight fields. The thread does not state the added size or decide on the split; P12 and P14 record no reason and add no support.

### 3. Each future skip reason adds four or eight more view columns

fixable · level 2, rubric item 5 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-2`

A later patch tracks autovacuum skips caused by another worker, as raised in message #29. Under the rule in message #30 it adds its own counters and timestamps. pg_stat_all_tables then holds sixteen or more skip columns, and dashboards that use select * change on each release.

The lock_skipped_ naming was chosen because more skip reasons are expected (messages #29, #30, #33). It fixes the pattern of four modes times two fields per reason in the main table view. Sami asked in message #6 whether vacuum columns should move to a dedicated view, and the thread does not answer. With ten patches like this one, the view would no longer be usable.

Thread: [#6 (Sami Imseih, 2026-03-25)](https://www.postgresql.org/message-id/CAA5RZ0v%3Dtdz3nGgq-Sk7hEwac1P%3D5J7d%2BSXd%2B62azfqiogdG6w%40mail.gmail.com), [#29 (Shinya Kato, 2026-07-11)](https://www.postgresql.org/message-id/CAOzEurSn7YDLLNnZW6etRJ167ZG4hvdaYDxeX5qwy%2BtuOFe4Dw%40mail.gmail.com), [#30 (Sami Imseih, 2026-07-21)](https://www.postgresql.org/message-id/CAA5RZ0txJaRyRnwWQiKEsOGYxU7FhKhX49qC5hZKPU7eSQz24Q%40mail.gmail.com), [#33 (Yugo Nagata, 2026-07-24)](https://www.postgresql.org/message-id/20260725014832.b25dcd5b1a8c44f2e8720f8d%40sraoss.co.jp)

**Question:** Should skip tracking go into a dedicated vacuum view or a single keyed form, so that a second skip reason does not add eight more columns to pg_stat_all_tables?

Counter-check: held. Message #30 says another skip reason would need its own counter, and message #6's question about a dedicated vacuum view has no answer. Sami does not expect a second reason, so the risk is moderate, but the layout question stays open.

### 4. Manual SKIP_LOCKED counters repeat a warning the caller already gets

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), security / privileges, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A maintenance script runs VACUUM (SKIP_LOCKED) and already receives "skipping vacuum of ... lock not available" as a WARNING. The four manual columns add only a second copy of that warning. They are also the path by which a role with no rights on a table can change its statistics, as message #51 shows.

Manual VACUUM and ANALYZE with SKIP_LOCKED already return a WARNING to the session that asked for the skip. The author first judged that warning sufficient (message #3) and added the manual columns at a reviewer's request (message #2). The motivating problem, autovacuum that tries and gets skipped, concerns only the autovacuum columns. The privilege report in message #51 has no answer yet, and it exists only on the manual path.

Thread: [#2 (Sami Imseih, 2026-03-24)](https://www.postgresql.org/message-id/CAA5RZ0snnePNW1NKGKh%2BNyJ1CY26T5F_6-tTq%2BBHWM2kj1fN1g%40mail.gmail.com), [#3 (Yugo Nagata, 2026-03-24)](https://www.postgresql.org/message-id/20260325012847.e026ba1860c07288efe3e97d%40sraoss.co.jp), [#51 (vignesh C, 2026-11-05)](https://www.postgresql.org/message-id/CALDaNm0%2BJ2dG5h3mX0AqqV%3DcO-vcZC0qXH_%2B7ypQ1TNWDVQhNQ%40mail.gmail.com)

**Question:** Which users need manual skips in a shared view when their script already receives the WARNING, and is that worth the unprivileged-write path from message #51?

Counter-check: held. Message #3 concedes the WARNING already reports manual skips, and the thread shows one reviewer request and no user report for the manual columns. A per-table view is more convenient than the WARNING, but the thread does not show that need is common.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The stats file format changes and needs a PGSTAT_FILE_FORMAT_ID bump. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Eight new pg_proc functions and eight new pg_stat_all_tables columns need a catversion bump. |
| ABI / extension API | **yes** | The patch adds eight fields to the relation stats entry declared in pgstat.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | Message #51 shows the skip is recorded before the vacuum privilege check. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Eight view columns and eight functions stay for good, and each new skip reason would add more. |
| performance on the hot path | **yes** | Every relation stats entry in shared memory, indexes included, grows by eight fields. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection already-exists-1: Manual SKIP_LOCKED counters repeat a warning the caller already gets |

## 5. Questions the author must answer before anyone reviews code

1. Will the next version check vacuum and analyze privileges before recording a skip, and does that change the existing WARNING for roles without privileges?
2. Which users need manual SKIP_LOCKED skips in a shared view when their session already receives the WARNING?
3. How much does each relation stats entry grow, and should the index and table stats split land before this patch?
4. Should skip tracking go into a dedicated vacuum view, so that a second skip reason does not add eight more columns to pg_stat_all_tables?
5. Should a lock-skipped analyze of a foreign table be recorded anywhere, given that pg_stat_all_tables does not show foreign tables?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier took part in the thread on the stats layout and backs the timestamps. The owner needs to know the cumulative statistics code and the lock and privilege order in vacuum.c.
- **Liability:** Eight view columns and eight functions that cannot be removed later, a larger relation stats entry for every table and index, and the handling of a concurrent drop in pgstat_relation.c.
- **Author follow-up:** The author posted twenty versions and answered each review round through message #50. Message #51 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users who monitor autovacuum under lock contention gain a per-table signal that only the server log gives today. The patch is about 46 KiB at v20 with three reviewers, but the privilege gap from message #51 must be closed before a committer can take it.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Eight columns land in pg_stat_all_tables while a stats split is still open. Refuted (no failure scenario): The patch puts the columns next to last_autovacuum and autovacuum_count, which is existing practice. A later move of vacuum columns would break queries on all of them equally, and no such move is proposed in the thread.
- `standards-compat-2` Counter meaning would change when the privilege check is added. Refuted (not specific): Duplicate of security-1. The changed counter meaning only arises if the privilege gap ships, and security-1 covers that gap.
- `demand-1` Manual-command skip columns add per-relation cost with little shown demand. Refuted (not specific): Duplicate of already-exists-1 for the manual-column demand, and of maintenance-fit-1 for the per-entry cost.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent has a recorded reason that still applies. P5's reason is not recorded, P7's use-case question is answered in messages #3 and #4, and the split attempts P12 and P14 have no recorded reason. The open privilege report in message #51 and the shared-memory concern in messages #6 and #9 belong to other angles.
- standards-compat: 2 objection(s). The naming, the SKIP_LOCKED-only scope of the manual columns, the VACUUM FULL exclusion and the parent-only recording for partitioned tables were discussed and settled in the thread, so they are not raised here.
- replication-ondisk: 0 objection(s). The only persistent state is the cumulative stats file and the relation stats entries in shared memory, both handled in the usual way. The stats file format needs a PGSTAT_FILE_FORMAT_ID bump at major-version upgrade; the commit message records it, and the stats file is not carried across pg_upgrade. The catalog side is new pg_proc entries and a pg_stat_all_tables change with a catversion bump. The proposal adds no WAL, no replication or wire-protocol behaviour, and no back-branch ABI change. The orphaned stats entry from a concurrent drop was found and fixed in messages #25 to #28 and #40.
- security: 1 objection(s). The autovacuum counters have no privilege aspect, and the new columns expose nothing that pg_stat_all_tables readers cannot already see.
- maintenance-fit: 2 objection(s). Two objections on memory and view growth. The privilege issue from message #51 belongs to another angle.
- already-exists: 1 objection(s). The autovacuum columns are only partly covered today. The skip message is logged only when log_autovacuum_min_duration is not -1, and reading it means parsing the server log. A per-table view is a real step beyond that, so this review raises no objection to those columns.
- demand: 1 objection(s). Demand for the autovacuum and autoanalyze columns looks real: three reviewers and a committer support it, and logs are the only current signal. The objection covers only the manual-command half.

### Precedents retrieved

- P1: [autovacuum may skip tables when session_authorization/role is set on database](https://commitfest.postgresql.org/patch/4702/) (cf, Returned with feedback, 2024-01-11) - judged not relevant; reason summarised from its thread
- P2: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P3: [Improve doc for autovacuum on partitioned tables](https://commitfest.postgresql.org/patch/3963/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P4: [autovacuum: add possibility to change priority of vacuumed tables](https://commitfest.postgresql.org/patch/1519/) (cf, Returned with feedback, 2018-03-02) - judged not relevant; reason summarised from its thread
- P5: [More stats about skipped/executed vacuums](https://commitfest.postgresql.org/patch/1351/) (cf, Returned with feedback, 2017-11-27) - used in the review; reason summarised from its thread
- P6: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - used in the review; reason summarised from its thread
- P7: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - used in the review; reason summarised from its thread
- P8: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P10: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - used in the review; reason summarised from its thread
- P11: [Autovacuum: vacuum tables in descending order of xid_age](https://commitfest.postgresql.org/patch/2364/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P12: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - used in the review
- P13: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P14: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - used in the review
- P15: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P16: [Revert "Show index search count in EXPLAIN ANALYZE."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d00107cd63e780753aa25563fa37603369997d0c) (revert, Reverted, 2025-03-05) - judged not relevant
- P17: [Switching XLog source from archive to streaming when primary available](https://commitfest.postgresql.org/patch/3663/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P18: [Revert "Blind attempt to fix _configthreadlocale() failures on MinGW."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a62d90f2e5cb55d0c8998b63120d6db3c55df866) (revert, Reverted, 2024-11-27) - judged not relevant
- P19: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant
- P20: [Allow reads to proceed during FK/trigger drops by reducing relation-level lock from AccessExclusive to ShareRowExclusive](https://commitfest.postgresql.org/patch/6124/) (cf, Withdrawn, 2025-11-25) - judged not relevant
- P21: [List of Available Figures/Graphics in the Documentation](https://commitfest.postgresql.org/patch/2204/) (cf, Returned with feedback, 2019-07-08) - judged not relevant
- P22: [Fix lock contention for HASHHDR.mutex](https://commitfest.postgresql.org/patch/442/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P23: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - judged not relevant
- P24: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 137 s; cost reported by the CLI: $3.42
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 86 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:57:16.081Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
