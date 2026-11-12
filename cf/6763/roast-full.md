# Design pre-review: Improve pg_stat_statements scalability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6763](https://commitfest.postgresql.org/patch/6763/) (Needs review); authors: Sami Imseih (simseih)
- Thread: [Improve pg_stat_statements scalability](https://www.postgresql.org/message-id/flat/CAA5RZ0vZwR_dSK6fo0P2-EnskUVN0NjLHnGnJMFDPC8-kEW3sQ%40mail.gmail.com), started 2026-05-11
- Messages read: 29 of 29
- Patch prose read: v4-0001-pgstat-Allow-pg_stat_force_next_flush-to-work-in-.patch, v4-0002-pgstat-allow-a-stats-kind-to-use-its-own-dedicate.patch, v4-0003-pgstat-add-pgstat_drop_current-for-use-in-dshash-.patch, v4-0004-WIP-pg_stat_statements-modernize-entry-storage-wi.patch, 0002-pg_stat_statements-modernize-entry-storage-with-pgst.patch, 0001-pgstat-add-pgstat_flush_pending-and-pg_stat_flush_pe.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and precedents P7, P3 and P4 each point to this rework as the preferred fix. Five fixable design points survive at level 2. The weightiest are what happens to the core stats file when the module is removed (replication-ondisk-1), whether pg_stat_statements.save = off still works (replication-ondisk-2), and the empty query column at the text cap (standards-compat-1). Confidence is medium because only 3 of 29 messages are visible and replication-ondisk-1 rests on reader behaviour not shown in the material.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the stats file reader's handling of an unregistered kind, the eviction and backstop logic, and the flush path cost at low client counts could not be checked. Message #28 is truncated, so some open items may already cover the questions above.

## The proposal, restated

**Problem.** pg_stat_statements does not scale under high concurrency. Every counter update takes a per-entry spinlock, and eviction at pg_stat_statements.max holds an exclusive LWLock that blocks all backends. Query text garbage collection rewrites the whole text file under that lock. Users with many cores or many distinct queries see lock convoys, and changing pg_stat_statements.max needs a restart.

**Proposed design.** The v5/v6 series stores statement entries as a custom pgstat kind in its own dshash, with counters accumulated per backend and merged on flush. Eviction becomes least-recently-used at about pg_stat_statements.max, with a hard backstop that stops new entries when allocated entries reach twice the max. A final patch moves query texts from the external file into capped shared memory.

**User-visible surface.**
- pg_stat_statements.max changes from server-start to PGC_SIGHUP and becomes a soft limit.
- New GUC pg_stat_statements.query_text_memory caps shared memory for query texts; the default moved from 64MB to 4MB during the thread.
- pg_stat_statements_info.dealloc now counts evicted entries, not eviction runs.
- pg_stat_force_next_flush() flushes non-transactional counters immediately when called inside a transaction.
- pgstat C API for extensions: own_hash kind option, pgstat_get_hash_for_kind(), and an option to release references after flush.

**What the author claims.**
- From 32 vCPUs up, 0004 gives 1.3x to 5x upstream TPS, close to a server without pg_stat_statements.
- Under extreme churn on 96 and 192 vCPUs, 0004 has 2.6x to 5.1x upstream TPS.
- Some low-client datapoints show 0004 1% to 4% slower than upstream.
- All popular statements are kept, and the backstop is never hit with the reference-release patch.
- Keeping texts in memory costs about what reading the text file already costs at each view query.

**Thread state.** Several infrastructure pieces are committed, and the in-transaction flush moved to its own thread, where reviewers asked for a flush-context flag and an explicit callback result. No reviewer has replied to v5/v6, and in-memory-only query texts drew doubts at PGConf.EU.

## 2. Prior attempts

### Cancel problems of query to pg_stat_statements (related)

- **Link:** [https://commitfest.postgresql.org/patch/5825/](https://commitfest.postgresql.org/patch/5825/)
- **Thread:** [https://www.postgresql.org/message-id/flat/31431745515032@mail.yandex-team.ru](https://www.postgresql.org/message-id/flat/31431745515032@mail.yandex-team.ru)
- **Date:** decided 2026-03-25; started 2025-04-24
- **Outcome:** Rejected (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Andrey Borodin first noted that the lock held during the file read blocks interrupts anyway; v2 worked around that. Michael Paquier then rejected the patch as only a workaround for a larger problem: the on-disk PGSS query text file does not scale once it is large and entries turn over often. He pointed instead to a planned rework that would move pg_stat_statements onto the pgstats API as a custom stats kind, with the query texts held entirely in memory.
- **Reason (quote):** "This patch is only a workaround for a larger problem: the PGSS text file does not and will never scale once it reaches a large size and once we have a high turnover rate of the PGSS entries due to different query IDs." ([message](https://www.postgresql.org/message-id/acNuZJJORIgbZ03a%40paquier.xyz))
- **What is different this time:** P7 patched cancellation of the query text file read. This proposal does the rework P7's reviewer asked for: a custom stats kind, and in the final patch query texts held in shared memory.

### Sampling in pg_stat_statements (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/5390/](https://commitfest.postgresql.org/patch/5390/)
- **Thread:** [https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com](https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com)
- **Date:** decided 2025-04-09; started 2024-11-18
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Ilia Evdokimov (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew it as Returned with Feedback, stating that the change was not a priority and that integrating pg_stat_statements with central pgstats was more important. Before that, the patch had been marked ready for committer, but no committer took it up. Alena Rybakina had objected that random sampling can drop the queries that matter.
- **Reason (quote):** "Since at the moment these changes are not a priority, and it's more important to address [0] for pg_stat_statements, I'm marking this patch as "Returned with Feedback"." ([message](https://www.postgresql.org/message-id/6983272b-9741-411e-b68a-b06fae2b52aa%40tantorlabs.com))
- **What is different this time:** P4 cut spinlock contention by random sampling, which can drop queries that matter. This proposal keeps full counts and removes the per-entry spinlock through backend-local pending stats, the pgstats integration P4 deferred to.

### Sampling of queries in pg_stat_statements (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/5389/](https://commitfest.postgresql.org/patch/5389/)
- **Thread:** [https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com](https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com)
- **Date:** decided 2024-11-19; started 2024-11-18
- **Outcome:** Withdrawn (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not say why the author withdrew the entry, one day after posting it. The only reservation came from Michael Paquier, who did not want to add sampling before studying the actual bottlenecks. He suggested moving pg_stat_statements into the central pgstats with the custom APIs, which would remove the query text file and shrink the hash key, but he did not plan that work for PG18.
- **Reason (quote):** "FWIW, I'm not eager to integrate this proposal without looking at this exact argument in depth." ([message](https://www.postgresql.org/message-id/Zz0MFPq1s4WFxxpL%40paquier.xyz))
- **What is different this time:** P3 also sampled queries to lower overhead. Its only reservation pointed to moving pg_stat_statements into central pgstats with the custom APIs, which this proposal does.

### Replace spinlock with lwlock in pg_stat_statements (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2634/](https://commitfest.postgresql.org/patch/2634/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2895b53b033c47ccb22972b589050dd9@EX13D05UWC001.ant.amazon.com](https://www.postgresql.org/message-id/flat/2895b53b033c47ccb22972b589050dd9@EX13D05UWC001.ant.amazon.com)
- **Date:** decided 2020-09-30; started 2020-06-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P12 swapped the per-entry spinlock for an LWLock. This proposal removes per-entry locking on the execution path. No reason is recorded for P12.

### Flush some statistics within running transactions (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/6389/](https://commitfest.postgresql.org/patch/6389/)
- **Thread:** [https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal](https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal)
- **Date:** decided 2026-03-25; started 2026-01-12
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** P9 proposed flushing statistics inside running transactions, as 0001 does. The thread moved 0001 to its own thread, where reviewers asked for a flush-context flag and an explicit callback result. No reason is recorded for P9.


## 3. Objections a reviewer is likely to raise

### 1. Removing pg_stat_statements could discard all cumulative statistics at restart

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `replication-ondisk-1`

An operator drops pg_stat_statements from shared_preload_libraries and restarts the server. At startup the stats file holds entries of a kind that is no longer registered, and pgstat may reject the whole file. Relation stats such as dead tuple counts reset, and autovacuum then works without them until activity rebuilds them.

Today pg_stat_statements writes its own file, so removing it leaves core stats untouched. With 0003 its entries are a custom kind written by the core pgstat machinery. As far as I know, the stats file reader treats an unknown kind as corruption and resets everything. The proposal does not say whether that path changes. pg_stat_statements is the most widely loaded custom kind, so this rare case would become common.

Thread: [#1 (Sami Imseih, 2026-05-11)](https://www.postgresql.org/message-id/CAA5RZ0vZwR_dSK6fo0P2-EnskUVN0NjLHnGnJMFDPC8-kEW3sQ%40mail.gmail.com), [#28 (Sami Imseih, 2026-11-11)](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com)

**Question:** What happens to relation and other core stats on restart when the stats file holds pg_stat_statements entries but the module is no longer loaded?

Counter-check: held. As far as I know, the stats file reader discards the whole file when it meets an entry of an unregistered kind, so the scenario is real. The visible material does not say what happens after the module is removed.

### 2. pg_stat_statements.save = off no longer stops entries being saved

fixable · level 2, rubric item 14 · design, on-disk format / pg_upgrade · `replication-ondisk-2`

A user sets pg_stat_statements.save = off so that statement statistics do not survive a restart. After a clean restart, the entries come back from the core stats file, while their query texts, which are auxiliary data, are not saved. The user then sees old counters, possibly with an empty query column.

Message #18 says that write_to_file is fixed when the kind is registered. So save = off stops only the auxiliary data from being written, and the entries are still written. Message #18 also lists a runtime override of write_to_file as missing infrastructure. Neither the v5 description in message #28 nor the committed pieces it lists mention that override.

Thread: [#18 (Sami Imseih, 2026-06-26)](https://www.postgresql.org/message-id/CAA5RZ0uyiH5Ke-AFqSn9R-g4wAQq7b4TbOBtNWKNZrfz%2BXqoRw%40mail.gmail.com), [#28 (Sami Imseih, 2026-11-11)](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com)

**Question:** In v5/v6, does pg_stat_statements.save = off still prevent entries from being written and restored, and how do entries and query texts stay consistent after restart?

Counter-check: held. Message #18 names the runtime override of write_to_file as missing, and the commits listed in message #28 do not include it. Message #28 is truncated, so a later open item may cover it, but nothing visible does.

### 3. In-memory query texts can leave the query column empty after upgrade

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A site with long ORM-generated queries upgrades and keeps the 4MB query_text_memory default. Once the cap fills, new entries show an empty query column. Monitoring snapshots that join on query text record entries they cannot label, and staff find this only when they need the text during an incident.

Today every tracked entry has its text, because the external file has no cap. In message #9, 0004 makes texts memory-only with a default of 4MB, and message #28 says real-world texts are much larger than the benchmark texts. No reviewer has replied on this, and message #28 reports doubts from PGConf.EU. The material does not say what users see when the cap is reached, or for how long.

Thread: [#9 (Sami Imseih, 2026-05-30)](https://www.postgresql.org/message-id/CAA5RZ0uoxiQ2_%3DxHGRnyc4WdM9aR0fzdMhBubnw97po%3D%3D--yGQ%40mail.gmail.com), [#28 (Sami Imseih, 2026-11-11)](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com)

**Question:** What does the query column return when query_text_memory is full, and how does an existing user learn the cap was reached?

Counter-check: held. Message #9 partly answers it: the query column is empty, and the text is filled in when the entry is next touched after the cap is raised. The proposal does not say how a user learns the cap was reached, and message #28 leaves the 4MB default open.

### 4. Release-after-flush option and stale-ref check fit only pg_stat_statements

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later pgstat change alters how entry references are counted or garbage-collected but tests only the default mode. Kinds that set the new release-at-flush option then keep dropped entries alive or free them early. pg_stat_statements hits its backstop and stops tracking new statements, and users find out from missing rows.

Message #28 introduces 0002 with no thread of its own and no tests, and it puts pgstat_entry_ref_is_stale() in pgstat_internal.h for one extension. Message #25 already objected that the series exposes dshash details to stats kinds and couples drop helpers to scan state. Reference lifetime then has two modes, and pgstat maintainers must keep both correct. Message #28 shows the 3.3 design depends on this option to stay below the backstop.

Thread: [#25 (Kyotaro Horiguchi, 2026-07-16)](https://www.postgresql.org/message-id/20260716.135423.1181957553378153013.horikyota.ntt%40gmail.com), [#28 (Sami Imseih, 2026-11-11)](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com)

**Question:** Which other stats kind would use release-at-flush, and what test fails if a future pgstat change breaks it?

Counter-check: held. Message #28 names no second user for release-at-flush and confirms 0002 has no thread of its own yet. Message #25 was about pgstat_drop_current() in v4 and must not be used as support here.

### 5. dealloc keeps its name but changes its meaning

fixable · level 2, rubric item 12 · design · `standards-compat-2`

A monitoring tool alerts on the rate of pg_stat_statements_info.dealloc, which until now counted eviction runs. After upgrade the same column counts evicted entries, which can be hundreds per run. Alerts fire or thresholds drift with no change in workload, and operators find the cause only by reading the release notes.

Message #9 states the change in semantics and asks for other views, and no one replied. The documentation hunk redefines the column in place. Postgres usually adds a new column when a counter's unit changes, so tools that read the old column keep working.

Thread: [#9 (Sami Imseih, 2026-05-30)](https://www.postgresql.org/message-id/CAA5RZ0uoxiQ2_%3DxHGRnyc4WdM9aR0fzdMhBubnw97po%3D%3D--yGQ%40mail.gmail.com)

**Question:** Why redefine dealloc in place instead of keeping it and adding a separate column for evicted entries?

Counter-check: held. Message #9 states that dealloc changes from eviction runs to evicted entries and asks for views, and no reviewer answered. The claim about usual Postgres practice has no support in the material, but the break for tools that read the column stands without it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Entries move from the module's own file into the core pgstat stats file, and the final patch removes the query text file. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series adds the own_hash kind option, pgstat_get_hash_for_kind() and a release-after-flush option to the pgstat API for extensions. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The series adds pg_stat_statements.query_text_memory and changes pg_stat_statements.max to PGC_SIGHUP with soft-limit semantics. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | pgstat gains a second reference-lifetime mode and a per-kind dshash that every later pgstat change must keep correct. |
| performance on the hot path | **yes** | Counter updates run on every tracked statement; the author reports some low-client datapoints 1% to 4% slower than upstream. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What happens to relation and other core stats at restart when the stats file holds pg_stat_statements entries but the module is no longer loaded?
2. In v5/v6, does pg_stat_statements.save = off still stop entries from being written and restored, and how do entries and query texts stay consistent after restart?
3. How does a user learn that query_text_memory is full, and which default keeps texts for workloads with long queries?
4. Which other stats kind would use the release-after-flush option, and which test fails if a later pgstat change breaks it?
5. Why redefine pg_stat_statements_info.dealloc in place instead of keeping it and adding a separate column for evicted entries?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Several pgstat infrastructure pieces from the series are already committed, so a committer who knows the cumulative statistics code is involved. No reviewer has replied to v5/v6 yet.
- **Liability:** The owner signs for the pgstat API additions used by extensions, the stats file behaviour with a widely loaded custom kind, and changed meanings of pg_stat_statements.max and dealloc, for as long as the module ships.
- **Author follow-up:** The author posted six versions over six months, split pieces into separate threads on request, and moved the entry through four CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** The gains appear from 32 vCPUs up and under query churn, in one of the most widely used contrib modules. The main patch is about 130 KiB and touches both pgstat core and pg_stat_statements, so review is costly, but the open points are fixable.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` Renaming the pg_stat_statements wait event breaks existing monitoring. Refuted (already handled): Message #28 raises the rename as an open question and states its cost to tools that watch the old name. Wait event names have changed between major versions before, and the eviction convoys that such a dashboard watches for are what the series removes.
- `security-1` Any role can fill query_text_memory and blank texts for all users. Refuted (not specific): Duplicate of standards-compat-1: the failure is the same empty query column at the cap. Upstream has no per-role limit either, so one role's churn can already evict other roles' entries, and message #28 reports the backstop is not reached with 0002.
- `maintenance-fit-2` Shared memory grows with a soft max and never shrinks. Refuted (no failure scenario): The peak is bounded by the backstop derived from the max the operator set, and the documentation hunk states that memory is reused but not returned. Upstream allocates memory for max at server start and also needs a restart to shrink it, so this is not a regression.
- `maintenance-fit-3` Removing the query text file commits all users to capped in-memory texts. Refuted (not specific): Duplicate of standards-compat-1, with the same scenario and the same question to the author. P7 records a committer pointing toward in-memory texts, so it does not support this objection, and message #28 says 0003 can go in alone with the file kept.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a reason that applies against this proposal. P7, P3 and P4 each name the move to a custom pgstats kind with in-memory texts as the preferred direction, and this series is that move. P9 and P12 have no recorded reason. The open questions about in-memory-only texts and the backstop belong to other angles.
- standards-compat: 3 objection(s). The SQL-level surface stays small, and making pg_stat_statements.max PGC_SIGHUP is a widely wanted change. The objections above are about existing view columns, wait events and text availability that change under unchanged names.
- replication-ondisk: 2 objection(s). Other persistent state looks acceptable. WAL is not touched. Standby behavior follows existing pgstat rules. pg_upgrade does not carry pg_stat files, so the old pg_stat_statements file format does not need to upgrade. The ABI changes affect only the master branch.
- security: 1 objection(s). The remaining privilege aspects look unchanged: view access, reset functions, and the pid-signalling flush from v0 were dropped. Without the code I could not check whether the new info columns or wait events expose anything new.
- maintenance-fit: 3 objection(s). Not applicable. The direction has committer support: message #17 and precedents P7, P3 and P4 favour moving pg_stat_statements to pgstats, so the objections above are about parts of the design.
- already-exists: 0 objection(s). No existing feature, setting or well-known extension removes the per-entry spinlock and the exclusive eviction lock that the thread measures. Raising pg_stat_statements.max needs a restart and does not touch hot-entry contention. The design reuses the core pgstat machinery instead of adding a second one. Earlier sampling and text-file workarounds [P3], [P4], [P7] were set aside in favour of this direction.
- demand: 0 objection(s). The material shows real, lasting demand. Message #1 cites a field complaint, several independent people back the goal (message #5: "Let's fix this for Postgres 20"), and the topic drew sessions at PGConf.dev and PGConf.EU. Rejections [P7], [P3] and [P4] each pointed to this rework as the preferred fix, and pg_stat_statements is one of the most widely used contrib modules.

### Precedents retrieved

- P1: [pg_stat_statements with plans (v02)](https://commitfest.postgresql.org/patch/1470/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P2: [pg_stat_statements with plans](https://commitfest.postgresql.org/patch/969/) (cf, Returned with feedback, 2017-03-04) - judged not relevant; reason summarised from its thread
- P3: [Sampling of queries in pg_stat_statements](https://commitfest.postgresql.org/patch/5389/) (cf, Withdrawn, 2024-11-19) - used in the review; reason summarised from its thread
- P4: [Sampling in pg_stat_statements](https://commitfest.postgresql.org/patch/5390/) (cf, Returned with feedback, 2025-04-09) - used in the review; reason summarised from its thread
- P5: [Sample values for pg_stat_statements](https://commitfest.postgresql.org/patch/1460/) (cf, Returned with feedback, 2019-01-31) - judged not relevant; reason summarised from its thread
- P6: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant; reason summarised from its thread
- P7: [Cancel problems of query to pg_stat_statements](https://commitfest.postgresql.org/patch/5825/) (cf, Rejected, 2026-03-25) - used in the review; reason summarised from its thread
- P8: [Improve cursor handling in pg_stat_statements](https://commitfest.postgresql.org/patch/3901/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P9: [Flush some statistics within running transactions](https://commitfest.postgresql.org/patch/6389/) (cf, Returned with feedback, 2026-03-25) - used in the review
- P10: [pg_stat_statements: Add gc_count and query_file_size to pgss_info](https://commitfest.postgresql.org/patch/6554/) (cf, Returned with feedback, 2026-03-25) - used in the review
- P11: [Revert "pg_stat_statements: Add coverage for entry_dealloc()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1141e29b61e9bf82b045540d41b27581fc3207ec) (revert, Reverted, 2023-12-31) - judged not relevant
- P12: [Replace spinlock with lwlock in pg_stat_statements](https://commitfest.postgresql.org/patch/2634/) (cf, Returned with feedback, 2020-09-30) - used in the review
- P13: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P14: [Allow index AMs to calculate custom statistics / BRIN](https://commitfest.postgresql.org/patch/3952/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P15: [fix spinlock contention in LogwrtResult](https://commitfest.postgresql.org/patch/2716/) (cf, Withdrawn, 2022-09-23) - judged not relevant
- P16: [CREATE SEQUENCE with RESTART option](https://commitfest.postgresql.org/patch/3069/) (cf, Rejected, 2021-07-29) - judged not relevant
- P17: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P18: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant
- P19: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P20: [pg_stat_statements and extended query protocol](https://commitfest.postgresql.org/patch/4146/) (cf, Withdrawn, 2023-01-30) - judged not relevant
- P21: [transaction statistics of logical replication subscription](https://commitfest.postgresql.org/patch/3504/) (cf, Returned with feedback, 2022-03-28) - judged not relevant
- P22: [Persist injection points across server restarts](https://commitfest.postgresql.org/patch/5731/) (cf, Returned with feedback, 2025-07-02) - judged not relevant
- P23: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant
- P24: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant

### Warnings

- tone lint: 2 finding(s) remain after the repair call; see tone_lint in roast.json before forwarding this text

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 232 s; cost reported by the CLI: $4.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 101 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 2 left
  - `brief.problem`: hyperbole ("garbage")
  - `objections[3].failure_scenario`: hyperbole ("garbage")
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:53.066Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
