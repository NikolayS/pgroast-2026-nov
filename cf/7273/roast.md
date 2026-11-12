# Design pre-review: Add a permission check to pg_stat_get_backend_subxact()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7273](https://commitfest.postgresql.org/patch/7273/); [thread](https://www.postgresql.org/message-id/flat/CAGRkXqTBZ%2BzbVuDC8xGEB6Btj61hsui5H5nGqzFBDyOXc%3D4bjQ%40mail.gmail.com), 27 of 27 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because v12 keeps a comment on pgstat_fetch_stat_backend_by_pid() about data from an older backend. The patch applies the existing HAS_PGSTAT_PERMISSIONS() rule to four functions, on HEAD only, after pgsql-security asked for a consistency fix and a committer agreed in message #3. Confidence is medium because only 3 of 27 messages and no code were read.

Not checked: Without the code, the handling of the stored role OID across stats reset, stats file reload and proc number reuse could not be checked. Whether the stats file format needs a version bump is also open.

## Prior attempts

- [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://www.postgresql.org/message-id/flat/3a60be45e7a89f50d166dba49553950d6b8a97f5.camel@cybertec.at) (related; Rejected, 2024-09-19). "Therefore, restricting EXPLAIN (ANALYZE) will do more harm than good, since it will make analyzing query performance harder without a real security gain." ([message](https://www.postgresql.org/message-id/ba677c20e02f554afb5a9aef8307cf4012eb4e63.camel%40cybertec.at)) Different now: P9 blocked a whole command for every unprivileged user. This patch applies the existing HAS_PGSTAT_PERMISSIONS() rule, so session owners and pg_read_all_stats members still see the numbers, as in pg_stat_activity.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which role OID does a PgStat_Backend entry hold after a stats reset, and for an entry read from a stats file written before the upgrade?
2. Do known monitoring tools call these four functions as a role without pg_read_all_stats, and does the change need a release note entry for them?
3. Is the race in pgstat_read_current_status() between the userid and the subxact counters accepted as existing behaviour, or is a follow-up planned?
4. Is a follow-up planned for snapshot reuse across other stats kinds, such as replication slot stats?

## Touch points

- **on-disk format / pg_upgrade:** PgStat_Backend gains a role OID field; whether the stats file format needs a version bump can only be checked in the code.
- **ABI / extension API:** The new role OID field changes the layout of PgStat_Backend in pgstat.h, which C extensions can see.
- **security / privileges:** Four functions start to hide session details from callers without pg_read_all_stats, superuser, or membership in the owning role.
- **backpatchability:** The output change for unprivileged callers is HEAD only; only the subxact_overflowed doc fix was backpatched.
- **maintenance burden ("who maintains this in 10 years"):** The subxact function checks the backend status entry role, while io, wal and lock check the role stored in PgStat_Backend.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier takes part in the thread, committed the doc fix and steered the design; the owner needs to know the cumulative statistics and backend status code.
- Priority: proceed; audience niche; review cost low. The patch is about 16 KiB, touches only the statistics code, docs and one regression test, and copies an existing rule. A committer who knows the area is engaged and a reviewer tested v12 without findings.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
