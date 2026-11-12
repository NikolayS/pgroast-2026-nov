# Design pre-review: Allow pg_read_all_stats to read replication origin status

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7183](https://commitfest.postgresql.org/patch/7183/); [thread](https://www.postgresql.org/message-id/flat/CAM6Zo8ywTSsi7JHEYLTbyvJ-ukH2Mr%3DZ611Qf4rQLhX12NyFqQ%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived against the goal or the demand. The one surviving objection, standards-compat-1, is fixable: pg_replication_origin_progress() would still deny the same LSN to the same role. The remaining choice between pg_read_all_stats and PUBLIC is a privilege-model decision, and no committer has spoken in the thread.

Not checked: The code was not read, and only 3 of 8 messages were shown. The exact grant statements and what the privileges.sql tests cover could not be checked.

## Prior attempts

None found before 2026-08-08.

## Objections a reviewer is likely to raise

### 1. Same origin progress stays superuser-only through pg_replication_origin_progress()

A monitoring tool running as a pg_monitor member reads remote_lsn for origin pg_16401 from pg_replication_origin_status. It then calls pg_replication_origin_progress('pg_16401', false) for the same value and gets "permission denied". The operator finds the gap only when the tool's second query fails in production.

After this patch, pg_replication_origin_status and pg_replication_origin_progress() expose the same per-origin LSN under different privilege rules, and the patch says nothing about the function. The brief lists this as open, and no message in the thread discusses it. A committer would likely ask for one rule for the read-only origin functions. Grants on write functions such as pg_replication_origin_advance() should stay as they are.

**Question:** Should pg_replication_origin_progress() also be granted to pg_read_all_stats, and if not, why should the same LSN need different privileges?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAM6Zo8ywTSsi7JHEYLTbyvJ-ukH2Mr%3DZ611Qf4rQLhX12NyFqQ%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAOh5eDUS77Q7FGG5tW9m5r9O0Z5ngzwthgQVFj7Jv8e5Tf2NOA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Should pg_replication_origin_progress() also be granted to pg_read_all_stats, and if not, why should the same LSN need different privileges?
2. Why is pg_read_all_stats the right holder of this access instead of PUBLIC, beyond the mixed precedent among replication views?
3. Does pg_replication_origin_session_progress() need a matching grant, or should it stay as it is?
4. Should the pg_read_all_stats description in user-manag.sgml change to mention pg_replication_origin_status?

## Touch points

- **catalog changes:** system_views.sql changes the default ACLs of pg_replication_origin_status and pg_show_replication_origin_status().
- **security / privileges:** Members of pg_read_all_stats and pg_monitor gain read access to origin names and LSNs that are superuser-only by default today.

## Who would commit it, and when

- Owner: silent. A committer who knows the predefined-role privilege model and logical replication origins would have to choose between pg_read_all_stats and PUBLIC.
- Priority: needs champion; audience niche; review cost low. The patch adds two grants and tests in three files, and it helps monitoring users who track replication origins. It waits on a committer's choice between pg_read_all_stats and PUBLIC.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
