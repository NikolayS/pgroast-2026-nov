# Design pre-review: Vacuum statistics

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5012](https://commitfest.postgresql.org/patch/5012/) (Needs review); authors: Andrei Lepikhov (lepikhov), Andrei Zubkov (zubkov), Alena Rybakina (a.rybakina)
- Thread: [Vacuum statistics](https://www.postgresql.org/message-id/flat/bee34abe-fe87-49b3-8523-8ae19e641ccf%40yandex.ru), started 2024-05-30
- Messages read: 161 of 161
- Patch prose read: v47-0001-Report-per-index-removed-tuples-in-vacuum-instru.patch, v47-0002-Track-vacuum-times-for-indexes-and-databases-and.patch, v47-0003-Count-wraparound-failsafe-vacuums-in-pg_stat-vie.patch, v47-0004-Count-vacuums-interrupted-by-errors-in-pg_stat_d.patch, v47-0005-Vacuum-report-hook-and-the-ext_vacuum_statistics.patch, v47-0006-ext_vacuum_statistics-page-counters-for-tables-a.patch, v47-0007-ext_vacuum_statistics-WAL-metrics-and-the-per-da.patch, v47-0008-ext_vacuum_statistics-shared-buffer-access-count.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Patches 0001-0004 draw no surviving objection, and the thread shows demand for them. The hook and contrib module in 0005-0008 fail at level 1b. Two reviewers asked which vacuum settings the counters would tune, and message #151 defers the answer (demand-1). The material also does not say why the storage module must ship in contrib (already-exists-2), and it cites no field report (demand-2).

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Five level 2 objections survived but were not evaluated because level 1b fails. Without the code, the GucContext of vacuum_statistics.enabled, hook chaining with object_access_hook, and the parallel accounting could not be checked.

## The proposal, restated

**Problem.** Vacuum work details (buffers, WAL, timing, dead and frozen tuples, pages) appear only in pg_stat_progress_vacuum while a vacuum runs, or once in the VACUUM VERBOSE or autovacuum log line. DBAs who tune autovacuum have no cumulative per-object view of vacuum workload over time.

**Proposed design.** The current v47 series has two parts. Patches 0001-0004 add a few columns to existing pg_stat views and a per-index removed-tuples count to the VERBOSE log line. Patches 0005-0008 add a core hook that receives a report after each table and index vacuum, plus a contrib extension that stores the reports as custom cumulative statistics; 0009 tracks clearing of visibility map bits.

**User-visible surface.**
- VACUUM VERBOSE and autovacuum log: per-index line gains "tuples: N removed".
- pg_stat_all_indexes, pg_stat_all_tables, pg_stat_database: total_(auto)vacuum_time and total_(auto)vacuum_delay_time columns where not already present.
- pg_stat_all_tables and pg_stat_database: vacuum_failsafe_count; pg_stat_database: vacuum_interrupt_count.
- C API: set_report_vacuum_hook, plus geterrlevel() in elog.h and visibilitymap_clear_rel().
- contrib ext_vacuum_statistics: pg_stats_vacuum_tables, pg_stats_vacuum_indexes and pg_stats_vacuum_database views, reset functions, and the vacuum_statistics.enabled GUC.

**What the author claims.**
- Without the extension loaded, the core overhead is a small stack struct and a NULL check on the hook.
- The heap report subtracts index buffer and WAL usage, so serial and parallel vacuum do not double-count.
- The log-line data is printed once and lost, so it cannot serve as queryable cumulative statistics.
- A dropped relation's entry is removed through object_access_hook, so OID reuse does not inherit old statistics.
- Patches 0001-0004 are useful on their own, apart from the hook and extension.

**Thread state.** Since 2024 the design moved from in-core views to an extension, back to core, then to the hook plus extension after Bharath and Amit argued most data exists in instrumentation and belongs in a module. Korotkov, Bharath, Karina and Vlada reviewed v42-v44; v47 has had no reply, and CFbot shows it needs a rebase.

## 2. Prior attempts

### Add last_vacuum_index_scans in pg_stat_all_tables (related)

- **Link:** [https://commitfest.postgresql.org/patch/3756/](https://commitfest.postgresql.org/patch/3756/)
- **Thread:** [https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com)
- **Date:** decided 2023-01-31; started 2022-07-04
- **Outcome:** Withdrawn (cf)
- **Who objected:** Alvaro Herrera, Fujii Masao (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Alvaro Herrera objected that the column keeps only the last vacuum's value, unlike other stats counters, which are all additive; he and Peter Geoghegan said a history of vacuum runs would be more useful. Fujii Masao asked whether one global counter in a pg_stat_vacuum view would be enough, and Alvaro later doubted the tuning value because autovacuum memory is capped at 1 GB. The author did not resolve these points, and the visible thread ends with a request for a rebased patch that got no reply before the entry was withdrawn.
- **Reason (quote):** "No other stats counter works that way; they are all additive." ([message](https://www.postgresql.org/message-id/20220708164052.bp62uhma56v7u7n5%40alvherre.pgsql))
- **What is different this time:** P1 kept only the last vacuum's index-scan count in one column. This proposal keeps additive cumulative counters, which answers the recorded objection, and puts most detail in an extension.


## 3. Objections a reviewer is likely to raise

### 1. No vacuum setting is named that the extension statistics would tune

fixable · level 1b, rubric item 2 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A DBA loads ext_vacuum_statistics and sees about 30 cumulative counters per table and index. No documented step leads from any counter to a change in autovacuum_vacuum_cost_limit, maintenance_work_mem or another setting, so the DBA watches the numbers and makes no tuning decision. The project then maintains a core hook and a contrib module that does not change what users do.

Amit asked in message #105 how these statistics would be used to tune vacuum, and Bharath repeated the question in message #150. In message #151 the author defers the answer and says it will come later. The stated goal in message #1 is tuning, so the case for 0005-0008 depends on that missing answer.

Thread: [#1 (Alena Rybakina, 2024-05-30)](https://www.postgresql.org/message-id/bee34abe-fe87-49b3-8523-8ae19e641ccf%40yandex.ru), [#105 (Amit Kapila, 2025-05-12)](https://www.postgresql.org/message-id/CAA4eK1JOUn%2BEqWSfRgKfgBZOXT7Q2dw2enmSZZgOhoMOFwopPA%40mail.gmail.com), [#150 (Bharath Rupireddy, 2026-07-13)](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#151 (Alena Rybakina, 2026-07-23)](https://www.postgresql.org/message-id/15a3a585-b55b-458e-be99-858238427329%40yandex.ru)

**Question:** For three or four counters in 0005-0008, which vacuum setting does each one change, and what does a DBA see before and after the change?

Counter-check: held. Messages #118 and #151 give some interpretations, such as delay time pointing to cost-based settings. But the author states in #151 that the mapping from statistics to settings will come later, so the question from #105 and #150 is still open.

### 2. Once the hook exists, the storage module need not be in contrib

fixable · level 1b, rubric item 16 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-2`

The project commits ext_vacuum_statistics with its views, reset functions, GUC and custom stats kinds, and then owns fixes to its parallel accounting, OID-reuse handling and privileges. Messages #152-#158 show such bugs found late. If someone starts the server once without the module loaded, all cumulative statistics are reset, and users report that to the project.

Bharath (message #150) and Amit (message #105) proposed core hooks with an external module, along the lines of pg_stat_statements, and the author has kept the module on GitHub since message #114. With set_report_vacuum_hook in core, the proposal does not say the module needs contrib for catalog access, grammar or performance. Custom cumulative statistics work out of tree, so contrib placement mainly moves the maintenance burden onto the project.

Thread: [#105 (Amit Kapila, 2025-05-12)](https://www.postgresql.org/message-id/CAA4eK1JOUn%2BEqWSfRgKfgBZOXT7Q2dw2enmSZZgOhoMOFwopPA%40mail.gmail.com), [#114 (Alena Rybakina, 2025-09-04)](https://www.postgresql.org/message-id/79490c8c-2c35-464d-bb3e-3d2ea5aca9ab%40postgrespro.ru), [#150 (Bharath Rupireddy, 2026-07-13)](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#152 (Alexander Korotkov, 2026-08-12)](https://www.postgresql.org/message-id/CAPpHfdud%3DYAi6Bnmzt4RgPmtshOJw9kHa9PLbfhBAbonJng7_g%40mail.gmail.com), [#157 (Vlada Pogozhelskaya, 2026-10-04)](https://www.postgresql.org/message-id/0234f0e0-ab7f-4397-963c-ae2c207da7e1%40gmail.com)

**Question:** What does ext_vacuum_statistics gain from contrib that an out-of-tree module using set_report_vacuum_hook cannot have?

Counter-check: held. pg_stat_statements shows contrib placement has precedent, and a hook benefits from an in-tree user for testing. But the material does not say why the full storage module, not a small test module, must ship in contrib.

### 3. Per-run vacuum data already exists in the log; demand for more is not shown

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-2`

Operators who need per-table vacuum history today set log_autovacuum_min_duration = 0 and aggregate the log lines. 0001-0004 already add totals, delay times and failsafe counts to pg_stat views. The extra buffer, WAL and page breakdowns in 0005-0008 serve a group whose size the thread does not show.

Message #150 maps almost every metric in the series to a field the VACUUM VERBOSE or autovacuum log already prints. The author answers in message #151 that log data is lost to rotation and needs a parser. That is an inconvenience with a working remedy. The interest in the thread comes from reviewers and patch authors (messages #9, #68). No user report from -general or -bugs is cited, and the thread does not show production numbers.

Thread: [#9 (Masahiko Sawada, 2024-06-27)](https://www.postgresql.org/message-id/CAD21AoDQ3bc%2BebbhMvDy9eZ%3DQBnN-mHTAc0569cbnWXjozx0dw%40mail.gmail.com), [#68 (Sami Imseih, 2025-01-02)](https://www.postgresql.org/message-id/CAA5RZ0sjDx6QFRJJ%2Be%3DorwvLboLWO42f2qF0_HbDq-sMq%3DtW4g%40mail.gmail.com), [#150 (Bharath Rupireddy, 2026-07-13)](https://www.postgresql.org/message-id/CALj2ACWoOFfc%2Bx4XEqL9RL2zoU%2BaSXiVbX2kopPQbUiBN%3D_1%2BA%40mail.gmail.com), [#151 (Alena Rybakina, 2026-07-23)](https://www.postgresql.org/message-id/15a3a585-b55b-458e-be99-858238427329%40yandex.ru)

**Question:** Which production cases, outside this thread, could not be diagnosed from the autovacuum log line plus the 0001-0004 columns, and what did those operators need instead?

Counter-check: held. Developers from several companies called the statistics useful (messages #9, #68, #108), which lowers confidence in this objection. Still, the thread cites no field report, and the log line covers most metrics except the per-index WAL and buffer breakdown.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The module writes custom kinds into the statistics file; a start without the module discards all cumulative statistics. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | 0002-0004 change system_views.sql and pg_proc.dat and bump catversion for new pg_stat columns. |
| ABI / extension API | **yes** | The series adds set_report_vacuum_hook and its report struct, geterrlevel() and visibilitymap_clear_rel(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The contrib module adds vacuum_statistics.enabled. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The module adds reset functions restricted to superusers by default, and views that show all relations. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Sampling code in vacuumlazy.c and vacuumparallel.c, a core hook and a contrib module all need upkeep. |
| performance on the hot path | **yes** | Vacuum samples buffer and WAL usage around the heap run and each index pass, and pgstat entries gain fields. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Reviewers argued the detailed counters belong in a module, and that module could live out of tree. |

## 5. Questions the author must answer before anyone reviews code

1. For three or four counters in 0005-0008, which vacuum setting does each one lead a DBA to change, and what does the DBA see before and after?
2. What does ext_vacuum_statistics gain from contrib that an out-of-tree module using set_report_vacuum_hook cannot have?
3. Which production cases could not be diagnosed from the autovacuum log line plus the 0001-0004 columns?
4. Can the statistics file reader skip entries of an unregistered custom kind, so that removing the module keeps the built-in counters?
5. Should all vacuum counters include or exclude index work in the table figure, and where is that rule documented for both sets of views?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows lazy vacuum, parallel vacuum and the cumulative statistics system. Reviewers commented on v42-v44, but v47 has no reply.
- **Liability:** The owner signs for the hook and its report struct as extension API, the index-usage subtraction in vacuum code, and a contrib module with custom statistics kinds, for as long as they ship.
- **Author follow-up:** Strong sign: the authors have posted 47 versions since 2024 and keep moving the entry between CommitFests.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A worked mapping from a few of the 0005-0008 counters to specific autovacuum settings, plus field reports of cases that the autovacuum log line and the 0001-0004 columns could not diagnose.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The nine patches touch vacuumlazy.c, vacuumparallel.c, pgstat and a new contrib module, and the design changed direction several times. Patches 0001-0004 could go ahead as their own entry, while 0005-0008 wait for the tuning answer and the decision on contrib placement.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) In-core columns and extension columns use different accounting rules. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-3` (fixable, level 2) Extension view and column names depart from cumulative statistics conventions. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Starting once without the module discards all built-in cumulative statistics. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Index-usage subtraction in vacuum code is fragile and fails silently. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Report hook is shaped for heap and for this one extension. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Tuning use of the statistics is still unstated after two years. Refuted (not specific): Duplicate of demand-1, which makes the same point from messages #105, #150 and #151 without relying on [P1]. [P1] concerned a last-value column, so it must not be used as support.
- `standards-compat-1` Starting once without the module resets all built-in cumulative statistics. Refuted (not specific): Duplicate of replication-ondisk-1, which states the same reset of built-in statistics and describes the effect on autovacuum more precisely.
- `security-1` User-settable vacuum_statistics.enabled lets sessions opt out of collection. Refuted (no failure scenario): The scenario depends on the GUC being user-settable, and the material does not show its context. The README `SET` example and per-database setting are both possible with a superuser-only GUC, so this is a question for code review.
- `maintenance-fit-1` Running once without the module resets all built-in cumulative statistics. Refuted (not specific): Duplicate of replication-ondisk-1. The item 4 framing also does not fit, because the reset affects only clusters that loaded the module.
- `already-exists-1` Hook report repeats the counters the vacuum log line already prints. Refuted (not specific): Duplicate of demand-2 for the log-line argument, and of demand-1 for the tuning question. The hook design itself came from the reviewers in messages #105 and #150, and a hook struct that lacks a new log counter is not a failure.

### Review angles

- prior-attempts: 1 objection(s). The only listed precedent close to this proposal, [P1], died on a non-additive counter, and this proposal fixes that. [P2] was closed for inactivity, which does not apply here. One objection is raised on the tuning question that [P1] and this thread both leave open.
- standards-compat: 3 objection(s). Three objections apply from the standards and compatibility angle. The VERBOSE log line format change is left out, because Postgres does not promise a stable log format.
- replication-ondisk: 1 objection(s). Other persistent state looks contained. pgstat file format and catversion bumps follow existing practice, cumulative stats are per node and not replicated as for built-in kinds, and drop and OID-reuse handling were settled in messages #152-#154.
- security: 1 objection(s). Other privilege points look settled or follow existing practice. Message #158 revoked the reset functions from PUBLIC. Like pg_stat_all_tables and pg_stat_database, the views show all relations and databases without privilege checks. Per-object memory growth matches built-in pgstat. The safety of hook chaining with object_access_hook and with sepgsql cannot be checked without the code.
- maintenance-fit: 3 objection(s). Patches 0001 to 0004 look like a reasonable first step on their own. These objections target the hook, the core sampling code and the contrib module.
- already-exists: 2 objection(s). Patches 0001-0004 add small columns that have no existing equivalent, so I raise nothing against them from this angle.
- demand: 2 objection(s). Patches 0001-0004 have demand shown in the thread. Sami asked for failsafe counts and vacuum time (messages #68, #75), and the delay and index time columns extend existing columns. The objections above concern only the hook and extension in 0005-0008.

### Precedents retrieved

- P1: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - used in the review; reason summarised from its thread
- P3: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant; reason summarised from its thread
- P4: [Add index scan progress to pg_stat_progress_vacuum](https://commitfest.postgresql.org/patch/3456/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P5: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant; reason summarised from its thread
- P6: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant
- P8: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant; reason summarised from its thread
- P9: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P10: [Extensible storage manager API - smgr hooks](https://commitfest.postgresql.org/patch/3216/) (cf, Returned with feedback, 2022-03-22) - judged not relevant
- P11: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P12: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant
- P13: [Allow index AMs to calculate custom statistics / BRIN](https://commitfest.postgresql.org/patch/3952/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P14: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant
- P15: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P16: [Introduce ProcessInterrupts_hook for C extensions](https://commitfest.postgresql.org/patch/2952/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P17: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P18: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P19: [PGDOCS - inconsistent linkends to "monitoring" views](https://commitfest.postgresql.org/patch/4602/) (cf, Rejected, 2023-11-30) - judged not relevant
- P20: [Waits monitoring](https://commitfest.postgresql.org/patch/297/) (cf, Returned with feedback, 2015-12-24) - judged not relevant
- P21: [estimating joins using extended statistics](https://commitfest.postgresql.org/patch/3055/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P22: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant
- P23: [Function to track shmem reinit time](https://commitfest.postgresql.org/patch/1575/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P24: [CLUSTER command progress monitor](https://commitfest.postgresql.org/patch/1268/) (cf, Returned with feedback, 2017-10-02) - judged not relevant

### Warnings

- attachment limit (8) reached; v47-0009-Track-table-VM-stability.patch not read
- attachment limit (8) reached; 0001-Machinery-for-grabbing-an-extended-vacuum-statistics.patch not read
- 44 long messages in the middle of the thread were left out of the prompts
- tone lint: 1 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 280 s; cost reported by the CLI: $9.28
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 120 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 1 left
  - `committability.owner_note`: about the author ("lazy")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:29:40.774Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
