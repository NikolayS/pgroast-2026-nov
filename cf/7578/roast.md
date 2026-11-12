# Design pre-review: Periodically refresh database frozen IDs during autovacuum

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7578](https://commitfest.postgresql.org/patch/7578/); [thread](https://www.postgresql.org/message-id/flat/CAPsk3_AMqBgVx4d1rAJJ%2Bi-U7TaQUqTkeAkN4P%2BhCfbRGCeCtA%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the core design. Message #1 reports a production database with over 600,000 tables, and a reviewer who knows the area called the current behavior problematic in message #4. Two fixable level 2 points remain: standards-compat-1 (vacuumdb gets no periodic refresh) and maintenance-fit-2 (the locking change is not a separate patch). The interval and its cost are still unmeasured.

Not checked: Without the code, the review could not check how often the progress counter is flushed or whether the varsup.c change keeps the existing lock order around the XID limit update. The pg_class scan cost at 600,000 tables is unmeasured.

## Prior attempts

None found before 2026-11-02.

## Objections a reviewer is likely to raise

### 1. vacuumdb runs with SKIP_DATABASE_STATS and gets no periodic refresh

A DBA near the XID stop limit runs `vacuumdb --jobs` over a database with 600,000 tables. Each per-table VACUUM skips database statistics, and the patch honors that skip, so datfrozenxid stays old until the final ONLY_DATABASE_STATS pass. The DBA finds this out when new XIDs are still refused hours after the oldest tables are frozen.

The commit message says the patch will "honor the option to skip database statistics updates". vacuumdb uses that option, so the periodic refresh covers manual multi-table VACUUM and autovacuum but not vacuumdb. vacuumdb is a common tool for manual wraparound recovery, and message #8 shows the gap in testing. v2 does not say whether vacuumdb will get its own refresh path or will keep its current behavior.

**Question:** How does a vacuumdb run that uses SKIP_DATABASE_STATS get datfrozenxid advanced before its final pass, and does the patch change that option's documented meaning?

<sub>fixable, level 2, design; messages [#7](https://www.postgresql.org/message-id/CAPsk3_ChAcWjuAVWVKuOCBdRKr8_m7FBeqPVKupcK_HYxqvmfg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAGRkXqQ9Umd0qxg63%2Bxxcj2zuQvF2Rpm%3Dj-FPvdsHHndhka0vQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Independent locking change still bundled with the refresh logic

A committer pushes v2 as one commit, and a later report shows datfrozenxid lagging or a VACUUM stalling on the database frozen-ID lock. Nobody can revert or bisect the conditional LockDatabaseFrozenIds change separately from the new periodic-refresh path, because both sit in one commit across vacuum.c, lmgr.c and varsup.c.

In message #4, Andres says non-blocking locking is independent of refreshing more often and 'probably should be a separate patch'. Message #7 adopts conditional locking but posts a single v2-0001, and the patch grew from 11.1 KB to 17.7 KB. The locking change also benefits the existing final refreshes, so it could be reviewed and committed first on its own.

**Question:** Can the conditional-locking change go in as a separate first patch, with the periodic refresh rebased on top of it?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/qxuq4rzul7sm3wuzsuo6i3ageo36twraucenusrfdiwzjboxne%40igohofmvqlfn), [#7](https://www.postgresql.org/message-id/CAPsk3_ChAcWjuAVWVKuOCBdRKr8_m7FBeqPVKupcK_HYxqvmfg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. How does a vacuumdb run that uses SKIP_DATABASE_STATS get datfrozenxid advanced before its final ONLY_DATABASE_STATS pass?
2. Can the conditional-locking change be posted as a separate first patch, with the periodic refresh rebased on top of it?
3. What does one vac_update_datfrozenxid() call cost in the 600,000-table database, and how does that compare with the ten-minute interval?
4. Why is ten minutes the right default interval, and will a GUC follow?
5. Does any view, such as pg_stat_database, show the new per-database freezing progress counter?

## Touch points

- **WAL / replication / standby:** datfrozenxid updates and the clog truncation that follows now run during a vacuum run, and both are WAL-logged.
- **catalog changes:** pg_database.datfrozenxid and datminmxid get updated more often; the patch adds no catalog column.
- **ABI / extension API:** The patch changes pgstat.h, vacuum.h and lmgr.h, including the shared per-database statistics entry.
- **new GUC:** The ten-minute interval is fixed in v2, and message #1 leaves a GUC for it undecided.
- **psql UX conventions:** vacuumdb uses SKIP_DATABASE_STATS, which v2 honors, so vacuumdb runs get no periodic refresh.
- **backpatchability:** The shared statistics entry change needs a stats file format bump; the CommitFest entry targets version 20.
- **maintenance burden ("who maintains this in 10 years"):** New refresh gating in vacuum(), a conditional database frozen-ID lock and a varsup.c change sit on the wraparound protection path.
- **performance on the hot path:** Each periodic refresh scans pg_class, which is large in a database with 600,000 tables; no cost measurement is shown.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund and Nathan Bossart take part in the thread; the owner needs to know vacuum, autovacuum and the XID limit code in varsup.c.
- Priority: proceed; audience a sizeable minority; review cost medium. Installations with very many tables near the XID stop limit benefit, and message #1 reports one. The 17.7 KB patch touches vacuum, autovacuum, lmgr, varsup and pgstat, and reviewers who know the area are already engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
