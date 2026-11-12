# Design pre-review: Report bytes and transactions actually sent downtream

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5877](https://commitfest.postgresql.org/patch/5877/) (Needs review); authors: Ashutosh Sharma (ashu_coek), Ashutosh Bapat (ashutoshbapat)
- Thread: [Report bytes and transactions actually sent downtream](https://www.postgresql.org/message-id/flat/CAExHW5s6KntzUyUoMbKR5dgwRmdV2Ay_2%2BAnTgYGAzo%3DQv61wA%40mail.gmail.com), started 2025-06-30
- Messages read: 123 of 123
- Patch prose read: v20260202-0001-Number-of-bytes-sent-downstream-by-logical.patch, v20260202-0002-Report-output-plugin-statistics-in-pg_stat.patch, 0001-Report-data-sent-statistics-in-pg_stat_repl-20250630.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated, and the thread shows demand and committer support for one additive counter kept by core. Two fixable level 2 points remain. standards-compat-1 shows that output_bytes counts the WALData header for walsender consumers but drops the LSN and XID columns for SQL API consumers. standards-compat-2 shows that the documented definition does not cover non-transactional logical messages. Both need a committer's decision on the column's definition before commit.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the cost of the increment in OutputPluginWrite() and the exact bytes counted in each output writer are not checked. The refactoring patch that adds replslot_stats.h was not shown and is not judged here.

## The proposal, restated

**Problem.** pg_stat_replication_slots reports total_txns and total_bytes, which count what enters the reorder buffer, so they look the same for slots with different publications. The author hit a lagging slot feeding Debezium and could not tell whether the walsender or the consumer was slow. Users need the amount of logical change data actually produced for the downstream.

**Proposed design.** The current patch adds one counter, kept by core, incremented in OutputPluginWrite() for the output plugin payload plus output-writer framing (the WALData header in the walsender path). Earlier designs with plugin-maintained counters, then bool-returning callbacks for filtered_bytes and sent_txns, were split off and set aside after Andres Freund objected. A second patch refactors the shared slot stats counters into a common struct.

**User-visible surface.**
- New column output_bytes (bigint) in pg_stat_replication_slots, earlier named sent_bytes.
- Changed DEBUG2 message from UpdateDecodingStats(), now with labelled fields.
- New header src/include/replication/replslot_stats.h with PgStat_ReplSlotStats, used by pgstat.h and reorderbuffer.h (refactoring patch).
- Set aside: filtered_bytes, sent_txns, plugin columns and bool return types for output plugin callbacks.

**What the author claims.**
- total_bytes cannot show the load on a downstream, because publication filtering and the output format change the volume.
- Protocol messages such as keepalives are not logical change data and do not load the consumer pipeline, so output_bytes excludes them.
- BEGIN/COMMIT and STREAM START/STOP messages belong to the logical output and are counted.
- The walsender WALData header is part of the output message given to libpq, so output_bytes includes it.
- The change only adds a column, so existing users and output plugins see no break.

**Thread state.** Masahiko Sawada, Amit Kapila and Michael Paquier accepted the name output_bytes and its scope. Aditya Gollamudi's question about counting the 25-byte walsender header got a doc change and a new test in the 2026-08-24 version, with no reply since; the refactoring patch still needs a separate agreement.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-06-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. One output_bytes column counts different framing per consumer path

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, wire protocol · `standards-compat-1`

An operator has two slots on the same publication. One is read by a walsender and the other by pg_logical_slot_get_changes(). The operator compares output_bytes and sees the walsender slot about 25 bytes higher per message for identical output, then reads that as more data produced.

Message #117 removed the SQL API's LSN and XID columns because they are not plugin output. Message #123 keeps the 25-byte WALData header in the walsender path as output-writer framing. Message #112 accepted the name because protocol-level bytes do not belong in slot stats, yet the WALData header is protocol framing. The column therefore has no single definition, and the documentation can only describe the difference.

Thread: [#112 (Masahiko Sawada, 2026-06-24)](https://www.postgresql.org/message-id/CAD21AoC%2BJoYKBAgTXR4fDWeL71cRDOvx-JmCOVQhWhCHG6r34w%40mail.gmail.com), [#114 (Amit Kapila, 2026-06-28)](https://www.postgresql.org/message-id/CAA4eK1KVQfQtdYKocfsxB1njEQGx4fhMX4N0bjmozG7Sx_V8Sw%40mail.gmail.com), [#117 (Ashutosh Sharma, 2026-06-30)](https://www.postgresql.org/message-id/CAE9k0PnBLCA0ZrzanQVyQCRnfk3rRZqctejCC3Cp9ZnhjMiyvQ%40mail.gmail.com), [#121 (Aditya Gollamudi, 2026-08-20)](https://www.postgresql.org/message-id/802ced07-1530-4cb3-9bf3-215fe61308ee%40gmail.com), [#123 (Ashutosh Sharma, 2026-08-24)](https://www.postgresql.org/message-id/CAE9k0PmtTYcayY5i3oDFazbZy0QqVYYiYqxP9FF-EY8RZcsj_g%40mail.gmail.com)

**Question:** Why should output_bytes include the WALData header for walsender consumers but exclude the LSN and XID columns for SQL API consumers, when both are framing added outside the plugin?

Counter-check: held. Message #123 confirms the walsender path counts the WALData header while the SQL API path drops the LSN and XID columns. The doc change describes the difference but nobody has replied to it, and message #116 asked for plugin-produced data only.

### 2. Counting of non-transactional logical messages is not settled in the contract

fixable · level 2, rubric item 13 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user emits non-transactional messages with pg_logical_emit_message(false, ...) and watches output_bytes. The value does not move until a later transaction triggers a stats update. If a later fix changes this behaviour or total_bytes, dashboards built on the first release change meaning without notice.

Message #89 shows that UpdateDecodingStats() does not run for non-transactional messages, so their bytes are not reported when they are sent. Message #91 calls this probably a bug but defers it. Message #114 defines output_bytes as transaction changes plus their delimiting messages and does not cover non-transactional messages. The first release of the column would then document one rule and behave by another.

Thread: [#89 (Ashutosh Sharma, 2026-03-16)](https://www.postgresql.org/message-id/CAE9k0PmNLURfRzdAXDWZ9-fG6kUjxaa0NCQ3kE8PkATFAKUq4A%40mail.gmail.com), [#91 (Amit Kapila, 2026-03-17)](https://www.postgresql.org/message-id/CAA4eK1LPf_CiBCH0wCv1OHMft1X0m2%3Dytih%2Bva65x3RvPgU7SA%40mail.gmail.com), [#95 (Amit Kapila, 2026-03-17)](https://www.postgresql.org/message-id/CAA4eK1KoOGfvbrX73z_wo8oGHx5Yb9KDJABr%3DJ4cRUay2Vdezw%40mail.gmail.com), [#98 (Ashutosh Sharma, 2026-03-18)](https://www.postgresql.org/message-id/CAE9k0PnmyO62ZhP2yX_9GWeVtOYtXmjLH3oLzEvzec-rsWyVDw%40mail.gmail.com), [#114 (Amit Kapila, 2026-06-28)](https://www.postgresql.org/message-id/CAA4eK1KVQfQtdYKocfsxB1njEQGx4fhMX4N0bjmozG7Sx_V8Sw%40mail.gmail.com)

**Question:** Are non-transactional logical messages counted in output_bytes, when do they show up in the view, and will a later fix to total_bytes change either column's meaning?

Counter-check: held. Messages #94 and #95 agreed that non-transactional messages should count, and message #98 promised a fix for the missing stats report, but no later message shows it. Late reporting alone is existing behaviour, so the open point is the documented definition.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The pgstat replication slot entry gains a counter, so the stats file format changes. |
| WAL / replication / standby | **yes** | The patch touches walsender.c, logical.c and reorderbuffer.c in the logical decoding output path. |
| wire protocol | **yes** | see objection standards-compat-1: One output_bytes column counts different framing per consumer path |
| catalog changes | **yes** | system_views.sql and pg_proc.dat change to add the output_bytes column to pg_stat_replication_slots. |
| ABI / extension API | **yes** | The refactoring patch adds replslot_stats.h and changes structs in pgstat.h and reorderbuffer.h that extensions include. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: One output_bytes column counts different framing per consumer path |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The counter's meaning depends on what each output writer puts in the buffer before the plugin writes. |
| performance on the hot path | **yes** | OutputPluginWrite() increments the counter for every logical decoding output message, and the thread has no cost numbers. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why does output_bytes include the WALData header for walsender consumers but exclude the LSN and XID columns for SQL API consumers, when the output writer adds both?
2. Are non-transactional logical messages counted in output_bytes, and when do they appear in the view?
3. Will a later fix to the stats update for non-transactional messages change the meaning of output_bytes or total_bytes?
4. How does an operator use output_bytes alone to tell a slow walsender from a slow consumer in the Debezium case from message #1?
5. What is the measured cost of the new counter in OutputPluginWrite() for a workload with many small changes?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Masahiko Sawada, Amit Kapila and Michael Paquier accepted the name and scope in the thread; none has replied to the 2026-08-24 version. The owner needs to know logical decoding and the cumulative statistics code.
- **Liability:** The owner signs for the documented definition of output_bytes, which monitoring tools will depend on for as long as the column exists, and for one more counter in the slot stats path.
- **Author follow-up:** Good sign: the author revised the patch for over a year, posted a version on 2026-08-24 answering review, and a second author joined the entry on 2026-09-30.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Users of logical replication with filtered publications or third-party consumers gain a counter that core maintains, and the thread already agreed on its name and scope. The patch touches walsender, logical decoding and pgstat code, and two definition points still need an answer.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Walsender framing makes output_bytes mean different things per consumer. Refuted (not specific): Duplicate of standards-compat-1. The claim that the header nearly doubles the count for small messages has no measurement in the material.
- `maintenance-fit-1` output_bytes counts walsender framing but not SQL API framing. Refuted (not specific): Duplicate of standards-compat-1. The added point about a future header change or a new output writer follows from the same undefined scope and has no concrete scenario of its own.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the earlier designs inside this thread do not leave an open reason. Andres Freund objected to plugin-side tracking and bool callbacks (messages #54, #72), and the current core counter in OutputPluginWrite() follows his advice in message #72 to track sent bytes only. The rename to total_wal_bytes was dropped in messages #79 to #83.
- standards-compat: 2 objection(s). The new column is additive. Dropping the total_bytes rename removed the compatibility break for existing monitoring queries, and the name output_bytes was agreed in the thread.
- replication-ondisk: 1 objection(s). The rest of the persistent state is routine: one counter in the pgstat replication slot entry with a stats file format bump, a view column, and a struct change that is new in a major release. Stats are per node, so standbys and synced slots are not affected.
- security: 0 objection(s). From the security angle there is nothing to object to. The current patch adds one cumulative bigint counter, output_bytes, to pg_stat_replication_slots and keeps it in core inside OutputPluginWrite(). Users who can read that view today already see the related volume counters (total_bytes, spill_bytes, stream_bytes), so the new column shows no new kind of data. It adds no new privilege, function, callback, GUC or memory that a user can grow. One counter could add a little to what the view reveals: with a row-filtered publication, output_bytes next to total_bytes hints at how much decoded data passed the filter. That is a coarse aggregate. The downstream consumer receives the data anyway, and no realistic attack follows from it. A committer might still check that the view keeps its current access rules. The plugin callback API changes that would have touched extension-facing behaviour were set aside in message #73.
- maintenance-fit: 1 objection(s). The current design adds one counter that core maintains, with a cheap increment per output write. Three committers who know the area agreed on the scope, and the plugin API change was dropped. The remaining point is fixable.
- already-exists: 0 objection(s). Postgres has no per-slot count of logical decoding output bytes today. pg_stat_replication reports WAL positions, and total_bytes counts reorder buffer input. Network monitoring, suggested in message #104, gives wire bytes per connection, not per slot and not only output data, as message #107 notes. I know of no contrib module or extension that does this, and a hook would cost more than one core counter.
- demand: 0 objection(s). The thread shows enough demand for a single additive column. A production Debezium incident motivated it (message #1), Andres Freund called sent bytes "by *far* the most useful statistic" (message #72), and Michael Paquier, Amit Kapila and Masahiko Sawada support it (messages #76, #114, #116). Kyotaro Horiguchi's question about wire bytes got an answer in message #107; deciding whether that answer is enough is a design matter for the other angles.

### Precedents retrieved

- P1: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant; reason summarised from its thread
- P2: [logical decoding and replication of sequences, take 2](https://commitfest.postgresql.org/patch/3823/) (cf, Withdrawn, 2024-06-21) - judged not relevant; reason summarised from its thread
- P3: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P4: [Error "initial slot snapshot too large" in create replication slot](https://commitfest.postgresql.org/patch/3445/) (cf, Withdrawn, 2024-01-16) - judged not relevant; reason summarised from its thread
- P5: [pglogical_output - a general purpose logical decoding output plugin](https://commitfest.postgresql.org/patch/418/) (cf, Returned with feedback, 2016-02-03) - judged not relevant; reason summarised from its thread
- P6: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1710/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P7: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/1862/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P8: [Add function pg_advance_replication_slot](https://commitfest.postgresql.org/patch/1128/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P9: [Row filtering for logical replication](https://commitfest.postgresql.org/patch/2270/) (cf, Returned with feedback, 2020-03-16) - judged not relevant; reason summarised from its thread
- P10: [pg_replslotdata - a tool for displaying replication slot information](https://commitfest.postgresql.org/patch/3425/) (cf, Rejected, 2022-01-17) - judged not relevant
- P11: [Forbid the use of invalidated physical slots in streaming replication.](https://commitfest.postgresql.org/patch/4686/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P12: [Logical Replication](https://commitfest.postgresql.org/patch/701/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P13: [expose confirmed_flush for replication slots](https://commitfest.postgresql.org/patch/295/) (cf, Rejected, 2015-07-08) - judged not relevant
- P14: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P15: [Logical decoding on standby](https://commitfest.postgresql.org/patch/788/) (cf, Returned with feedback, 2017-11-24) - judged not relevant
- P16: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P17: [Track replay lag in pg_stat_replication](https://commitfest.postgresql.org/patch/920/) (cf, Returned with feedback, 2017-02-01) - judged not relevant
- P18: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P19: [Revert "Logical decoding of sequences"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2c7ea57e56ca5f668c32d4266e0a3e45b455bef5) (revert, Reverted, 2022-04-07) - judged not relevant
- P20: [Filter irrelevant change before reassemble transactions during logical decoding](https://commitfest.postgresql.org/patch/5592/) (cf, Withdrawn, 2025-02-21) - judged not relevant
- P21: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant
- P22: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P23: [multivariate statistics](https://commitfest.postgresql.org/patch/120/) (cf, Returned with feedback, 2015-08-25) - judged not relevant
- P24: [Revert "Detect setlocale(LC_CTYPE, NULL) clobbering previous return values."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f2a378de41bf3e516b8d2c4d65790aeefbfb89d) (revert, Reverted, 2015-06-20) - judged not relevant

### Warnings

- 5 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 120 s; cost reported by the CLI: $7.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 67 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:36:11.062Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
