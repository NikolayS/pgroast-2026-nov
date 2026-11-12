# Design pre-review: Track skipped vacuum and analyze in pg_stat_all_tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6639](https://commitfest.postgresql.org/patch/6639/); [thread](https://www.postgresql.org/message-id/flat/20260324151133.7940a5c1f2ebd594d54da481%40sraoss.co.jp), 51 of 51 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, and no surviving objection is fatal. The decisive point is security-1: message #51 shows that a role without privileges on a table can bump its skip counters, and v20 does not fix it. The other surviving points (already-exists-1, maintenance-fit-1, maintenance-fit-2) are fixable scope and layout questions that a committer who knows the cumulative statistics code can settle.

Not checked: Without the code, this review could not check the order of the lock attempt, the privilege check and the stats report in vacuum.c. It also could not check the syscache lookup that guards against a concurrent drop, or the real size of the stats entry.

## Prior attempts

- [More stats about skipped/executed vacuums](https://www.postgresql.org/message-id/flat/20171031.184937.57727308.horiguchi.kyotaro@lab.ntt.co.jp) (similar idea; Returned with feedback, 2017-11-27). The visible messages do not show why. Only the author's repost of the patch is shown, with no review or reply. (AI summary of the thread) Different now: P5 added many vacuum outcome columns, including a successive-failure count, and took a lock on each table inside the view. This proposal counts only lock skips and computes nothing at view time.
- [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://www.postgresql.org/message-id/flat/CAECtzeXXuMkw-RVGTWvHGOJsmFdsRY+jK0ndQa80sw46y2uvVQ@mail.gmail.com) (related; Returned with feedback, 2024-12-11). "I think the question to which we don't have a clear answer is: for what purpose would you want to be able to distinguish parallel and non-parallel scans on a per-table basis?" ([message](https://www.postgresql.org/message-id/CA%2BTgmoZ5oiVFF1%3DAqmESrJo4YPbk9WO8n%2BNnLPCm6wNrnoEooQ%40mail.gmail.com)) Different now: P7 added per-table counters without a clear per-table use. This proposal states its use: telling a table that autovacuum keeps skipping from one it never tries.

## Objections a reviewer is likely to raise

### 1. Roles without privileges on a table can bump its lock-skip statistics

A role with only USAGE on the schema runs `vacuum (skip_locked) t` in a loop while the owner's batch job holds a lock. The lock_skipped_vacuum_count of t grows and last_lock_skipped_vacuum moves forward. A DBA alerting on these columns sees false spikes or loses the real signal, and cannot tell who caused them.

Message #51 shows the skip is recorded before vacuum_is_permitted_for_relation(), and includes a test that reproduces it. Message #42 reported the same gap and suggested the RangeVarGetRelidExtended() callback. Message #43 left it for a later cleanup that "someone" may take on, and v20 still has the gap with no reply to #51. Today a role without privileges on t can affect only its own WARNING output. With this patch it writes to shared per-table statistics that other roles read and alert on.

**Question:** Will v21 check vacuum and analyze privileges before recording a skip, and does that check change the existing WARNING for roles without privileges?

<sub>fixable, level 2, security / privileges; messages [#42](https://www.postgresql.org/message-id/CAOzEurRddqm-etFkG9T4auovYUDdB1bBasNJGg1jW09tXcOmTg%40mail.gmail.com), [#43](https://www.postgresql.org/message-id/CAN12%2BYLDPuW8iF%3DUqXyLzmjM9MkHbx2YwwKq%2B8G2-HcO8Kzyfg%40mail.gmail.com), [#51](https://www.postgresql.org/message-id/CALDaNm0%2BJ2dG5h3mX0AqqV%3DcO-vcZC0qXH_%2B7ypQ1TNWDVQhNQ%40mail.gmail.com); `security-1`</sub>

### 2. Every index and table stats entry pays for eight vacuum-only fields

A site with hundreds of thousands of relations upgrades. Each PgStat_StatTabEntry in shared memory grows by eight 8-byte fields, for indexes too, though an index never records a lock skip. The site pays this memory whether or not anyone reads the new columns.

Message #7 notes that PgStat_StatTabEntry is shared by indexes and tables, and that the vacuum and analyze fields apply only to tables. Michael proposes a separate index stats kind as a start (messages #7, #9), but v20 does not do that split and does not state the added size. Earlier entries proposing that split were returned ([P12], [P14]), with no reason recorded, so the split is not assured to land first. The cost falls on all relations, but the gain goes only to tables under lock contention.

**Question:** How much does each relation stats entry grow, and should the index and table stats split land before this patch adds eight table-only fields?

<sub>fixable, level 2, performance on the hot path; messages [#7](https://www.postgresql.org/message-id/acRvIt2idOKeD7wv%40paquier.xyz), [#9](https://www.postgresql.org/message-id/acSMZ-0K_YUrmzQG%40paquier.xyz); `maintenance-fit-1`</sub>

### 3. Each future skip reason adds four or eight more view columns

A later patch tracks autovacuum skips caused by another worker, as raised in message #29. Under the rule in message #30 it adds its own counters and timestamps. pg_stat_all_tables then holds sixteen or more skip columns, and dashboards that use select * change on each release.

The lock_skipped_ naming was chosen because more skip reasons are expected (messages #29, #30, #33). It fixes the pattern of four modes times two fields per reason in the main table view. Sami asked in message #6 whether vacuum columns should move to a dedicated view, and the thread does not answer. With ten patches like this one, the view would no longer be usable.

**Question:** Should skip tracking go into a dedicated vacuum view or a single keyed form, so that a second skip reason does not add eight more columns to pg_stat_all_tables?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/CAA5RZ0v%3Dtdz3nGgq-Sk7hEwac1P%3D5J7d%2BSXd%2B62azfqiogdG6w%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAOzEurSn7YDLLNnZW6etRJ167ZG4hvdaYDxeX5qwy%2BtuOFe4Dw%40mail.gmail.com), [#30](https://www.postgresql.org/message-id/CAA5RZ0txJaRyRnwWQiKEsOGYxU7FhKhX49qC5hZKPU7eSQz24Q%40mail.gmail.com), [#33](https://www.postgresql.org/message-id/20260725014832.b25dcd5b1a8c44f2e8720f8d%40sraoss.co.jp); `maintenance-fit-2`</sub>

### 4. Manual SKIP_LOCKED counters repeat a warning the caller already gets

A maintenance script runs VACUUM (SKIP_LOCKED) and already receives "skipping vacuum of ... lock not available" as a WARNING. The four manual columns add only a second copy of that warning. They are also the path by which a role with no rights on a table can change its statistics, as message #51 shows.

Manual VACUUM and ANALYZE with SKIP_LOCKED already return a WARNING to the session that asked for the skip. The author first judged that warning sufficient (message #3) and added the manual columns at a reviewer's request (message #2). The motivating problem, autovacuum that tries and gets skipped, concerns only the autovacuum columns. The privilege report in message #51 has no answer yet, and it exists only on the manual path.

**Question:** Which users need manual skips in a shared view when their script already receives the WARNING, and is that worth the unprivileged-write path from message #51?

<sub>fixable, level 1b, desirability; messages [#2](https://www.postgresql.org/message-id/CAA5RZ0snnePNW1NKGKh%2BNyJ1CY26T5F_6-tTq%2BBHWM2kj1fN1g%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/20260325012847.e026ba1860c07288efe3e97d%40sraoss.co.jp), [#51](https://www.postgresql.org/message-id/CALDaNm0%2BJ2dG5h3mX0AqqV%3DcO-vcZC0qXH_%2B7ypQ1TNWDVQhNQ%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Will the next version check vacuum and analyze privileges before recording a skip, and does that change the existing WARNING for roles without privileges?
2. Which users need manual SKIP_LOCKED skips in a shared view when their session already receives the WARNING?
3. How much does each relation stats entry grow, and should the index and table stats split land before this patch?
4. Should skip tracking go into a dedicated vacuum view, so that a second skip reason does not add eight more columns to pg_stat_all_tables?
5. Should a lock-skipped analyze of a foreign table be recorded anywhere, given that pg_stat_all_tables does not show foreign tables?

## Touch points

- **on-disk format / pg_upgrade:** The stats file format changes and needs a PGSTAT_FILE_FORMAT_ID bump.
- **catalog changes:** Eight new pg_proc functions and eight new pg_stat_all_tables columns need a catversion bump.
- **ABI / extension API:** The patch adds eight fields to the relation stats entry declared in pgstat.h.
- **security / privileges:** Message #51 shows the skip is recorded before the vacuum privilege check.
- **maintenance burden ("who maintains this in 10 years"):** Eight view columns and eight functions stay for good, and each new skip reason would add more.
- **performance on the hot path:** Every relation stats entry in shared memory, indexes included, grows by eight fields.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection already-exists-1: Manual SKIP_LOCKED counters repeat a warning the caller already gets

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier took part in the thread on the stats layout and backs the timestamps. The owner needs to know the cumulative statistics code and the lock and privilege order in vacuum.c.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users who monitor autovacuum under lock contention gain a per-table signal that only the server log gives today. The patch is about 46 KiB at v20 with three reviewers, but the privilege gap from message #51 must be closed before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
