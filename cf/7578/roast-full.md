# Design pre-review: Periodically refresh database frozen IDs during autovacuum

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7578](https://commitfest.postgresql.org/patch/7578/) (Needs review); authors: Jingtang Zhang (mrdrivingduck)
- Thread: [\[PATCH\] Periodically refresh database frozen IDs during autovacuum](https://www.postgresql.org/message-id/flat/CAPsk3_AMqBgVx4d1rAJJ%2Bi-U7TaQUqTkeAkN4P%2BhCfbRGCeCtA%40mail.gmail.com), started 2026-11-02
- Messages read: 8 of 8
- Patch prose read: v2-0001-Periodically-refresh-database-frozen-IDs-during-VACUUM.patch, v1-0001-Periodically-refresh-database-frozen-IDs-during-a.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the core design. Message #1 reports a production database with over 600,000 tables, and a reviewer who knows the area called the current behavior problematic in message #4. Two fixable level 2 points remain: standards-compat-1 (vacuumdb gets no periodic refresh) and maintenance-fit-2 (the locking change is not a separate patch). The interval and its cost are still unmeasured.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check how often the progress counter is flushed or whether the varsup.c change keeps the existing lock order around the XID limit update. The pg_class scan cost at 600,000 tables is unmeasured.

## The proposal, restated

**Problem.** Vacuum and autovacuum update a database's datfrozenxid only after finishing the whole table list. In a database with over 600,000 tables, the database horizon can stay old long after the oldest tables are frozen. The XID stop limit uses the oldest datfrozenxid, so the instance can keep refusing new XIDs.

**Proposed design.** v2 counts relfrozenxid and relminmxid advances in shared per-database statistics and calls vac_update_datfrozenxid() between tables, in the common vacuum() path for autovacuum and manual multi-table VACUUM. A periodic refresh needs pending progress, ForceTransactionIdLimitUpdate() true and a fixed ten-minute interval elapsed. It takes the database frozen-ID lock conditionally and skips if busy; the existing final refreshes stay.

**User-visible surface.**
- Shared database statistics gain a per-database counter of table freezing progress; the material does not say whether any view shows it.
- datfrozenxid in pg_database can advance during a long VACUUM or autovacuum run, before it finishes.
- The refresh interval is fixed at ten minutes; the author has not decided on a GUC.

**What the author claims.**
- With 1,000 tables, three workers and a ten-second interval, database age fell with table age, updating about every ten seconds; without the patch it stayed flat until about 150 seconds.
- Manual multi-table VACUUM, tables ordered oldest first, showed the same behavior.
- The interval batches progress and cuts repeated pg_class scans in databases with many tables.
- Moving the fallback refresh to DROP TABLE or DROP DATABASE would add DDL latency and need care around rollback.

**Thread state.** Andres Freund asked for coverage of manual vacuum, conditional calls and non-blocking locking; v2 answers these, and Nathan Bossart's scoring question was a misunderstanding. Open are the interval and its cost, Andres's suggestion to split the locking change into a separate patch, and shihao zhong's question about vacuumdb.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-02; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. vacuumdb runs with SKIP_DATABASE_STATS and gets no periodic refresh

fixable · level 2, rubric item 13 · design, psql UX conventions, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A DBA near the XID stop limit runs `vacuumdb --jobs` over a database with 600,000 tables. Each per-table VACUUM skips database statistics, and the patch honors that skip, so datfrozenxid stays old until the final ONLY_DATABASE_STATS pass. The DBA finds this out when new XIDs are still refused hours after the oldest tables are frozen.

The commit message says the patch will "honor the option to skip database statistics updates". vacuumdb uses that option, so the periodic refresh covers manual multi-table VACUUM and autovacuum but not vacuumdb. vacuumdb is a common tool for manual wraparound recovery, and message #8 shows the gap in testing. v2 does not say whether vacuumdb will get its own refresh path or will keep its current behavior.

Thread: [#7 (Jingtang Zhang, 2026-11-08)](https://www.postgresql.org/message-id/CAPsk3_ChAcWjuAVWVKuOCBdRKr8_m7FBeqPVKupcK_HYxqvmfg%40mail.gmail.com), [#8 (shihao zhong, 2026-11-10)](https://www.postgresql.org/message-id/CAGRkXqQ9Umd0qxg63%2Bxxcj2zuQvF2Rpm%3Dj-FPvdsHHndhka0vQ%40mail.gmail.com)

**Question:** How does a vacuumdb run that uses SKIP_DATABASE_STATS get datfrozenxid advanced before its final pass, and does the patch change that option's documented meaning?

Counter-check: held. vacuumdb sends per-table VACUUM with SKIP_DATABASE_STATS on Postgres 16 and later, and the v2 commit message says the patch honors that option. Message #8 raises the gap and the thread has no answer yet.

### 2. Independent locking change still bundled with the refresh logic

fixable · level 2 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A committer pushes v2 as one commit, and a later report shows datfrozenxid lagging or a VACUUM stalling on the database frozen-ID lock. Nobody can revert or bisect the conditional LockDatabaseFrozenIds change separately from the new periodic-refresh path, because both sit in one commit across vacuum.c, lmgr.c and varsup.c.

In message #4, Andres says non-blocking locking is independent of refreshing more often and 'probably should be a separate patch'. Message #7 adopts conditional locking but posts a single v2-0001, and the patch grew from 11.1 KB to 17.7 KB. The locking change also benefits the existing final refreshes, so it could be reviewed and committed first on its own.

Thread: [#4 (Andres Freund, 2026-11-02)](https://www.postgresql.org/message-id/qxuq4rzul7sm3wuzsuo6i3ageo36twraucenusrfdiwzjboxne%40igohofmvqlfn), [#7 (Jingtang Zhang, 2026-11-08)](https://www.postgresql.org/message-id/CAPsk3_ChAcWjuAVWVKuOCBdRKr8_m7FBeqPVKupcK_HYxqvmfg%40mail.gmail.com)

**Question:** Can the conditional-locking change go in as a separate first patch, with the periodic refresh rebased on top of it?

Counter-check: held. In message #4 a reviewer who knows the area asked for the locking change as a separate patch, and message #7 posts one patch without answering. The "benefits the existing final refreshes" point is weak, because v2 keeps those refreshes blocking.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | datfrozenxid updates and the clog truncation that follows now run during a vacuum run, and both are WAL-logged. |
| wire protocol | no |  |
| catalog changes | **yes** | pg_database.datfrozenxid and datminmxid get updated more often; the patch adds no catalog column. |
| ABI / extension API | **yes** | The patch changes pgstat.h, vacuum.h and lmgr.h, including the shared per-database statistics entry. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The ten-minute interval is fixed in v2, and message #1 leaves a GUC for it undecided. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | vacuumdb uses SKIP_DATABASE_STATS, which v2 honors, so vacuumdb runs get no periodic refresh. |
| security / privileges | no |  |
| backpatchability | **yes** | The shared statistics entry change needs a stats file format bump; the CommitFest entry targets version 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | New refresh gating in vacuum(), a conditional database frozen-ID lock and a varsup.c change sit on the wraparound protection path. |
| performance on the hot path | **yes** | Each periodic refresh scans pg_class, which is large in a database with 600,000 tables; no cost measurement is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How does a vacuumdb run that uses SKIP_DATABASE_STATS get datfrozenxid advanced before its final ONLY_DATABASE_STATS pass?
2. Can the conditional-locking change be posted as a separate first patch, with the periodic refresh rebased on top of it?
3. What does one vac_update_datfrozenxid() call cost in the 600,000-table database, and how does that compare with the ten-minute interval?
4. Why is ten minutes the right default interval, and will a GUC follow?
5. Does any view, such as pg_stat_database, show the new per-database freezing progress counter?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund and Nathan Bossart take part in the thread; the owner needs to know vacuum, autovacuum and the XID limit code in varsup.c.
- **Liability:** The owner signs for the timing of datfrozenxid updates and clog truncation during vacuum runs, the conditional database frozen-ID lock, and the fixed interval. All of it sits on the wraparound protection path, so the liability is long-lived.
- **Author follow-up:** The author posted v2 in message #7, which answers the requests from message #4 for manual VACUUM coverage, conditional calls and non-blocking locking.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Installations with very many tables near the XID stop limit benefit, and message #1 reports one. The 17.7 KB patch touches vacuum, autovacuum, lmgr, varsup and pgstat, and reviewers who know the area are already engaged.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Wraparound refresh gating stored in lossy cumulative statistics. Refuted (no failure scenario): After a crash or reset, the next relfrozenxid advance adds to the counter again, so periodic refreshes resume while vacuum continues. The worst case is one delayed refresh, with the final refresh kept as today's fallback.
- `already-exists-1` Periodic VACUUM (ONLY_DATABASE_STATS) already refreshes datfrozenxid on demand. Refuted (already handled): In message #4 Andres Freund calls refreshing datfrozenxid only at the end "problematic" for autovacuum and multi-relation vacuum in general. A scheduled `vacuum (only_database_stats)` needs an outside scheduler for each database, while autovacuum is the built-in wraparound protection.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P7] sorted autovacuum tables by xid age and was returned over scheduling regressions; this proposal changes no table order, and Andres confirmed in message #5 that it adds no rescoring. [P12] concerns freezing strategy and records no reason. Message #1 builds on a committed prioritization change, not a rejected one.
- standards-compat: 1 objection(s). There is no other contract objection. Advancing datfrozenxid earlier is a benign behavior change, and the fixed interval adds no GUC. The proposal does not yet say whether the new per-database counter shows up in pg_stat_database.
- replication-ondisk: 0 objection(s). No objection from this angle. The persistent effect is the existing in-place update of pg_database.datfrozenxid and the clog truncation that follows, now run every ten minutes. Both are WAL-logged and replay on a standby as they do today. The new counter is shared statistics: it is not on disk across a crash, not replicated and not in the catalog. If the counter is lost or reset, the periodic refresh does not run and the final refresh still does, so behaviour falls back to today's. The PgStat_StatDBEntry change needs a stats file format bump, which only blocks backpatching, and the CommitFest entry targets version 20 anyway. Without the code I could not check how often the counter is flushed, or whether the varsup.c change keeps the existing locking order around the XID limit update.
- security: 0 objection(s). No security or privilege objection. Any user who may run a multi-table VACUUM already triggers vac_update_datfrozenxid() at the end of the command, so the patch only changes when that refresh runs. The conditional lock and the ten-minute gate do not add new ways to block others or use resources. The patch does not say whether a view exposes the new per-database statistics counter, so that exposure could not be checked.
- maintenance-fit: 2 objection(s). From this angle the hot-path cost is small, since there is one counter update per vacuumed table and refreshes are gated by ForceTransactionIdLimitUpdate(), and the idea fits core. Committers who know vacuum are engaged in the thread.
- already-exists: 1 objection(s). One objection only; no extension or external tool beyond the built-in ONLY_DATABASE_STATS option covers this.
- demand: 0 objection(s). The material shows real demand: a production database with over 600,000 tables where datfrozenxid stayed stale after the oldest tables were frozen (message #1). In message #4, Andres Freund called the update-only-at-end behavior problematic because it makes anti-wraparound measures more likely, and asked for the same logic in manual multi-relation VACUUM. Wraparound protection matters to every large installation, and the extra work in v2 runs only inside vacuum and only when progress is pending, so the users who gain also pay. The open points are the interval, its cost and vacuumdb coverage, which other angles cover.

### Precedents retrieved

- P1: [autovacuum: add possibility to change priority of vacuumed tables](https://commitfest.postgresql.org/patch/1519/) (cf, Returned with feedback, 2018-03-02) - judged not relevant; reason summarised from its thread
- P2: [autovacuum may skip tables when session_authorization/role is set on database](https://commitfest.postgresql.org/patch/4702/) (cf, Returned with feedback, 2024-01-11) - judged not relevant; reason summarised from its thread
- P3: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P4: [Improve doc for autovacuum on partitioned tables](https://commitfest.postgresql.org/patch/3963/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P5: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P6: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P7: [Autovacuum: vacuum tables in descending order of xid_age](https://commitfest.postgresql.org/patch/2364/) (cf, Returned with feedback, 2020-08-03) - used in the review; reason summarised from its thread
- P8: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P9: [Fix autovacuum freeze bug where weight can lower a table's freeze score](https://commitfest.postgresql.org/patch/7242/) (cf, Withdrawn, 2026-09-03) - judged not relevant; reason summarised from its thread
- P10: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P11: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant; reason summarised from its thread
- P12: [New strategies for freezing, advancing relfrozenxid early](https://commitfest.postgresql.org/patch/3843/) (cf, Withdrawn, 2023-01-26) - used in the review
- P13: [Fix a race condition in ConditionVariableTimedSleep()](https://commitfest.postgresql.org/patch/5736/) (cf, Returned with feedback, 2026-10-02) - judged not relevant
- P14: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [Improve recovery performance of dropping database having many tablespaces](https://commitfest.postgresql.org/patch/1658/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P16: [Revert: Allow locking updated tuples in tuple_update() and tuple_delete()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=193e6d18e553a104d315ff81892d509d89a30fd8) (revert, Reverted, 2024-04-11) - judged not relevant
- P17: [log_autovacuum_min_duration doesn't log VACUUMs](https://commitfest.postgresql.org/patch/1007/) (cf, Returned with feedback, 2017-03-07) - judged not relevant
- P18: [Add autovacuum_warning to surface concurrent vacuum collisions](https://commitfest.postgresql.org/patch/6830/) (cf, Withdrawn, 2026-08-22) - judged not relevant
- P19: [Introduce some randomness to autovacuum](https://commitfest.postgresql.org/patch/5732/) (cf, Rejected, 2026-06-07) - judged not relevant
- P20: [Revert "Increase the default vacuum_cost_limit from 200 to 2000"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=52985e4fea75f1ec742742f27e246a8775c99e08) (revert, Reverted, 2019-03-10) - judged not relevant
- P21: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant
- P22: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant
- P23: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P24: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 146 s; cost reported by the CLI: $1.47
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 65 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:01:39.574Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
