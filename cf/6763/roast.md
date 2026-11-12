# Design pre-review: Improve pg_stat_statements scalability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6763](https://commitfest.postgresql.org/patch/6763/); [thread](https://www.postgresql.org/message-id/flat/CAA5RZ0vZwR_dSK6fo0P2-EnskUVN0NjLHnGnJMFDPC8-kEW3sQ%40mail.gmail.com), 29 of 29 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and precedents P7, P3 and P4 each point to this rework as the preferred fix. Five fixable design points survive at level 2. The weightiest are what happens to the core stats file when the module is removed (replication-ondisk-1), whether pg_stat_statements.save = off still works (replication-ondisk-2), and the empty query column at the text cap (standards-compat-1). Confidence is medium because only 3 of 29 messages are visible and replication-ondisk-1 rests on reader behaviour not shown in the material.

Not checked: Without the code, the stats file reader's handling of an unregistered kind, the eviction and backstop logic, and the flush path cost at low client counts could not be checked. Message #28 is truncated, so some open items may already cover the questions above.

## Prior attempts

- [Cancel problems of query to pg_stat_statements](https://www.postgresql.org/message-id/flat/31431745515032@mail.yandex-team.ru) (related; Rejected, 2026-03-25). "This patch is only a workaround for a larger problem: the PGSS text file does not and will never scale once it reaches a large size and once we have a high turnover rate of the PGSS entries due to different query IDs." ([message](https://www.postgresql.org/message-id/acNuZJJORIgbZ03a%40paquier.xyz)) Different now: P7 patched cancellation of the query text file read. This proposal does the rework P7's reviewer asked for: a custom stats kind, and in the final patch query texts held in shared memory.
- [Sampling in pg_stat_statements](https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com) (similar idea; Returned with feedback, 2025-04-09). "Since at the moment these changes are not a priority, and it's more important to address [0] for pg_stat_statements, I'm marking this patch as "Returned with Feedback"." ([message](https://www.postgresql.org/message-id/6983272b-9741-411e-b68a-b06fae2b52aa%40tantorlabs.com)) Different now: P4 cut spinlock contention by random sampling, which can drop queries that matter. This proposal keeps full counts and removes the per-entry spinlock through backend-local pending stats, the pgstats integration P4 deferred to.
- [Sampling of queries in pg_stat_statements](https://www.postgresql.org/message-id/flat/fe99e0ca-e564-480e-b865-5f0cee30bc60@tantorlabs.com) (similar idea; Withdrawn, 2024-11-19). "FWIW, I'm not eager to integrate this proposal without looking at this exact argument in depth." ([message](https://www.postgresql.org/message-id/Zz0MFPq1s4WFxxpL%40paquier.xyz)) Different now: P3 also sampled queries to lower overhead. Its only reservation pointed to moving pg_stat_statements into central pgstats with the custom APIs, which this proposal does.
- [Replace spinlock with lwlock in pg_stat_statements](https://www.postgresql.org/message-id/flat/2895b53b033c47ccb22972b589050dd9@EX13D05UWC001.ant.amazon.com) (similar idea; Returned with feedback, 2020-09-30). reason not recorded Different now: P12 swapped the per-entry spinlock for an LWLock. This proposal removes per-entry locking on the execution path. No reason is recorded for P12.
- [Flush some statistics within running transactions](https://www.postgresql.org/message-id/flat/aWTVEycKj7Qh%2FSXH@ip-10-97-1-34.eu-west-3.compute.internal) (similar idea; Returned with feedback, 2026-03-25). reason not recorded Different now: P9 proposed flushing statistics inside running transactions, as 0001 does. The thread moved 0001 to its own thread, where reviewers asked for a flush-context flag and an explicit callback result. No reason is recorded for P9.

## Objections a reviewer is likely to raise

### 1. Removing pg_stat_statements could discard all cumulative statistics at restart

An operator drops pg_stat_statements from shared_preload_libraries and restarts the server. At startup the stats file holds entries of a kind that is no longer registered, and pgstat may reject the whole file. Relation stats such as dead tuple counts reset, and autovacuum then works without them until activity rebuilds them.

Today pg_stat_statements writes its own file, so removing it leaves core stats untouched. With 0003 its entries are a custom kind written by the core pgstat machinery. As far as I know, the stats file reader treats an unknown kind as corruption and resets everything. The proposal does not say whether that path changes. pg_stat_statements is the most widely loaded custom kind, so this rare case would become common.

**Question:** What happens to relation and other core stats on restart when the stats file holds pg_stat_statements entries but the module is no longer loaded?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#1](https://www.postgresql.org/message-id/CAA5RZ0vZwR_dSK6fo0P2-EnskUVN0NjLHnGnJMFDPC8-kEW3sQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. pg_stat_statements.save = off no longer stops entries being saved

A user sets pg_stat_statements.save = off so that statement statistics do not survive a restart. After a clean restart, the entries come back from the core stats file, while their query texts, which are auxiliary data, are not saved. The user then sees old counters, possibly with an empty query column.

Message #18 says that write_to_file is fixed when the kind is registered. So save = off stops only the auxiliary data from being written, and the entries are still written. Message #18 also lists a runtime override of write_to_file as missing infrastructure. Neither the v5 description in message #28 nor the committed pieces it lists mention that override.

**Question:** In v5/v6, does pg_stat_statements.save = off still prevent entries from being written and restored, and how do entries and query texts stay consistent after restart?

<sub>fixable, level 2, design; messages [#18](https://www.postgresql.org/message-id/CAA5RZ0uyiH5Ke-AFqSn9R-g4wAQq7b4TbOBtNWKNZrfz%2BXqoRw%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com); `replication-ondisk-2`</sub>

### 3. In-memory query texts can leave the query column empty after upgrade

A site with long ORM-generated queries upgrades and keeps the 4MB query_text_memory default. Once the cap fills, new entries show an empty query column. Monitoring snapshots that join on query text record entries they cannot label, and staff find this only when they need the text during an incident.

Today every tracked entry has its text, because the external file has no cap. In message #9, 0004 makes texts memory-only with a default of 4MB, and message #28 says real-world texts are much larger than the benchmark texts. No reviewer has replied on this, and message #28 reports doubts from PGConf.EU. The material does not say what users see when the cap is reached, or for how long.

**Question:** What does the query column return when query_text_memory is full, and how does an existing user learn the cap was reached?

<sub>fixable, level 2, design; messages [#9](https://www.postgresql.org/message-id/CAA5RZ0uoxiQ2_%3DxHGRnyc4WdM9aR0fzdMhBubnw97po%3D%3D--yGQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com); `standards-compat-1`</sub>

### 4. Release-after-flush option and stale-ref check fit only pg_stat_statements

A later pgstat change alters how entry references are counted or garbage-collected but tests only the default mode. Kinds that set the new release-at-flush option then keep dropped entries alive or free them early. pg_stat_statements hits its backstop and stops tracking new statements, and users find out from missing rows.

Message #28 introduces 0002 with no thread of its own and no tests, and it puts pgstat_entry_ref_is_stale() in pgstat_internal.h for one extension. Message #25 already objected that the series exposes dshash details to stats kinds and couples drop helpers to scan state. Reference lifetime then has two modes, and pgstat maintainers must keep both correct. Message #28 shows the 3.3 design depends on this option to stay below the backstop.

**Question:** Which other stats kind would use release-at-flush, and what test fails if a future pgstat change breaks it?

<sub>fixable, level 2, ABI / extension API; messages [#25](https://www.postgresql.org/message-id/20260716.135423.1181957553378153013.horikyota.ntt%40gmail.com), [#28](https://www.postgresql.org/message-id/CAN12%2BYLZ4KY0c_zb5i1Zc2Zi_3LoQvswCRE6T6AUx95_4Xn1dg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 5. dealloc keeps its name but changes its meaning

A monitoring tool alerts on the rate of pg_stat_statements_info.dealloc, which until now counted eviction runs. After upgrade the same column counts evicted entries, which can be hundreds per run. Alerts fire or thresholds drift with no change in workload, and operators find the cause only by reading the release notes.

Message #9 states the change in semantics and asks for other views, and no one replied. The documentation hunk redefines the column in place. Postgres usually adds a new column when a counter's unit changes, so tools that read the old column keep working.

**Question:** Why redefine dealloc in place instead of keeping it and adding a separate column for evicted entries?

<sub>fixable, level 2, design; messages [#9](https://www.postgresql.org/message-id/CAA5RZ0uoxiQ2_%3DxHGRnyc4WdM9aR0fzdMhBubnw97po%3D%3D--yGQ%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What happens to relation and other core stats at restart when the stats file holds pg_stat_statements entries but the module is no longer loaded?
2. In v5/v6, does pg_stat_statements.save = off still stop entries from being written and restored, and how do entries and query texts stay consistent after restart?
3. How does a user learn that query_text_memory is full, and which default keeps texts for workloads with long queries?
4. Which other stats kind would use the release-after-flush option, and which test fails if a later pgstat change breaks it?
5. Why redefine pg_stat_statements_info.dealloc in place instead of keeping it and adding a separate column for evicted entries?

## Touch points

- **on-disk format / pg_upgrade:** Entries move from the module's own file into the core pgstat stats file, and the final patch removes the query text file.
- **ABI / extension API:** The series adds the own_hash kind option, pgstat_get_hash_for_kind() and a release-after-flush option to the pgstat API for extensions.
- **new GUC:** The series adds pg_stat_statements.query_text_memory and changes pg_stat_statements.max to PGC_SIGHUP with soft-limit semantics.
- **maintenance burden ("who maintains this in 10 years"):** pgstat gains a second reference-lifetime mode and a per-kind dshash that every later pgstat change must keep correct.
- **performance on the hot path:** Counter updates run on every tracked statement; the author reports some low-client datapoints 1% to 4% slower than upstream.

## Who would commit it, and when

- Owner: qualified committer engaged. Several pgstat infrastructure pieces from the series are already committed, so a committer who knows the cumulative statistics code is involved. No reviewer has replied to v5/v6 yet.
- Priority: proceed; audience a sizeable minority; review cost high. The gains appear from 32 vCPUs up and under query churn, in one of the most widely used contrib modules. The main patch is about 130 KiB and touches both pgstat core and pg_stat_statements, so review is costly, but the open points are fixable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
