# Design pre-review: Unlogged materialized views

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7167](https://commitfest.postgresql.org/patch/7167/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [Unlogged materialized views](https://www.postgresql.org/message-id/flat/CAN4CZFO74UCnbu_nY-5vxju8p6_YKzUdOfhLVUuAGqwVMbCZHA%40mail.gmail.com), started 2026-07-20
- Messages read: 12 of 12
- Patch prose read: v6-0001-Add-durable-unlogged-reset-LSN-to-pg_control.patch, v6-0003-Add-unlogged-materialized-views.patch, v6-0002-Replace-pg_class.relispopulated-with-epoch-capabl.patch, 0001-Add-durable-unlogged-reset-generation-counter.patch, 0003-Add-unlogged-materialized-views.patch, 0002-Replace-pg_class.relispopulated-with-epoch-capable-r.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, and the LSN epoch answers the old blocker of catalog writes during recovery. Three design points are still open. replication-ondisk-1 shows a path where pg_upgrade can mark an empty unlogged matview as populated. standards-compat-1 shows that removing relispopulated breaks clients on clusters that never use the feature. standards-compat-2 is a pg_dump rule the thread left open.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check what binary upgrade writes for a stale stamp, or how pg_resetwal behaves on a standby shut down in recovery. It also could not check the cost of the populated check on matview access.

## The proposal, restated

**Problem.** REFRESH MATERIALIZED VIEW on large views is slow and writes a lot of WAL. The author targets users whose materialized views need not survive a crash, such as caches, rollups and lookup tables that can be rebuilt. Earlier threads agreed such views should become unpopulated after a crash, but pg_class cannot be updated during recovery.

**Proposed design.** pg_class.relispopulated (bool) becomes relpopulated (int8): 0 is unpopulated, 1 is crash-safe populated, any other value is an epoch stamp for an unlogged matview. pg_control gains a durable LSN marking where WAL ended at the last unlogged reset. A stamp older than the current epoch reads as unpopulated, so crash, promotion and PITR invalidate the view with no catalog write during recovery.

**User-visible surface.**
- create unlogged materialized view and alter materialized view ... set { logged | unlogged }.
- pg_class.relispopulated removed and replaced by relpopulated int8, so queries that read relispopulated fail.
- New function pg_matview_is_populated(regclass), which pg_matviews.ispopulated now uses.
- pg_control gains an unlogged-reset LSN, shown by pg_controldata and set by pg_resetwal.
- psql \d shows unlogged matviews as unlogged; pg_dump --no-unlogged-table-data does not apply to matviews.

**What the author claims.**
- Unlogged matviews avoid WAL during populate and refresh, which makes refresh considerably faster; the thread gives no numbers.
- Crash recovery, standby, promotion and PITR behave as expected; recovery TAP tests cover crash, standby and promotion.
- An LSN-based epoch cannot falsely match a stale stamp, while a reset counter plus timeline could, as when a standby starts without standby.signal.
- pg_upgrade keeps unlogged contents because pg_resetwal sets the epoch only when the reset loses data.
- Two users replied +1 and listed typical rebuildable matview uses.

**Thread state.** Jim Jones reviewed twice, fixed docs, tab completion and \d, flagged the removal of pg_class.relispopulated for client tools such as pgAdmin, and raised error message points; v6 addresses the error messages. No committer has replied, and the author still wants agreement on the approach before adding auto-refresh.

## 2. Prior attempts

### Detectable crashes and unlogged table resets (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3183/](https://commitfest.postgresql.org/patch/3183/)
- **Thread:** [https://www.postgresql.org/message-id/flat/62750df5b126e1d8ee039a79ef3cc64ac3d47cd5.camel@j-davis.com](https://www.postgresql.org/message-id/flat/62750df5b126e1d8ee039a79ef3cc64ac3d47cd5.camel@j-davis.com)
- **Date:** decided 2022-03-07; started 2021-06-03
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Discussion stopped after June 2021 with open questions: whether LSN or timestamp is useful (Tom Lane doubted LSN), whether to reset the counter, and whether to track promotions. The entry moved through four commitfests with no further activity, and Michael Paquier closed it because it had been idle too long.
- **Reason (quote):** "This is too long, so let's discard this patch for now." ([message](https://www.postgresql.org/message-id/YiM8fhx2udJJwlJ7%40paquier.xyz))
- **What is different this time:** P9 recorded the last crash recovery in pg_control so applications could detect unlogged resets, with no consumer in core. This proposal stores a similar LSN but uses it inside core to invalidate unlogged matviews, and it also covers promotion and PITR.


## 3. Objections a reviewer is likely to raise

### 1. pg_upgrade may mark a crash-emptied unlogged matview as populated

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, catalog changes, pg_dump / restore round-trip · `replication-ondisk-1`

A PG20 cluster crashes and recovery resets an unlogged matview to its empty init fork, but nobody refreshes it. The user then runs pg_upgrade to PG21, and the new cluster marks the matview populated while the transferred file is empty. Queries return no rows without any error, and the user may only notice from wrong reports.

Patch 0002 says pg_dump reads `relpopulated <> 0` and, in binary upgrade, calls binary_upgrade_set_matview_populated(). A stale epoch stamp is non-zero, so the dump treats it as populated. The old stamp is an LSN from the old cluster's WAL, so it means nothing in the new cluster, whose epoch starts at the initdb bootstrap checkpoint. Copying the stamp would invalidate every unlogged matview on upgrade, and re-stamping would validate empty ones. The pg_upgrade claim in the brief covers only the pg_resetwal step and does not address this case.

Thread: [#7 (Zsolt Parragi, 2026-09-24)](https://www.postgresql.org/message-id/CAN4CZFOLPsqmDoyU_BX8%3D%3DQx4_cvF5c4CZb-K3aUn2OESdDCjw%40mail.gmail.com), [#12 (Zsolt Parragi, 2026-10-15)](https://www.postgresql.org/message-id/CAN4CZFOrHgLMBngZE21CsfC_Mqj4vbeRdkZouUW6dNb1mu2f3Q%40mail.gmail.com)

**Question:** During pg_upgrade, what does binary_upgrade_set_matview_populated() write for an unlogged matview whose old-cluster stamp is stale, and which test covers that case?

Counter-check: held. The 0002 commit message says pg_dump reads `relpopulated <> 0`, and a stale stamp is non-zero. The 0001 text covers only the pg_resetwal step, so the material does not say what binary upgrade writes for a stale unlogged matview.

### 2. Removing pg_class.relispopulated breaks every client for a narrow feature

fixable · level 2, rubric item 4 · catalog changes, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A user upgrades to the new release and opens pgAdmin or a monitoring script that reads c.relispopulated. The query fails with "column c.relispopulated does not exist", even though the cluster has no unlogged matview. The user finds out at the first catalog query after the upgrade.

Message #9 shows pgAdmin failing, and message #11 notes it is not the only tool affected. The author says in message #10 that the column cannot stay because it would go stale. That is true only for unlogged matviews. A bool column could stay correct for every existing relation, with the epoch held in a separate column, so the cost would fall only on users of the new feature. The new int8 column also mixes a flag with an LSN typed as int8, and a natural rewrite such as relpopulated <> 0 reports a stale unlogged matview as populated.

Thread: [#9 (Jim Jones, 2026-10-13)](https://www.postgresql.org/message-id/bd706289-ce95-41bf-b417-595ec80168f8%40uni-muenster.de), [#10 (Zsolt Parragi, 2026-10-14)](https://www.postgresql.org/message-id/CAN4CZFO7NREnzNPkAmyeWRtMtAp6WPR1PqSp7wQUCb7yB--8Fg%40mail.gmail.com), [#11 (Jim Jones, 2026-10-15)](https://www.postgresql.org/message-id/999c81c2-77ca-4903-883b-e01a31d12aab%40uni-muenster.de)

**Question:** Why can't relispopulated stay as a bool that is exact for all existing relations, with the unlogged epoch kept in a separate column or in pg_matview_is_populated() only?

Counter-check: held. Message #10 explains why the bool would go stale, but only for unlogged matviews, and the thread does not discuss a separate epoch column. Catalog columns do change between major versions, so this stays fixable.

### 3. pg_dump --no-unlogged-table-data skips unlogged tables but not unlogged matviews

fixable · level 2, rubric item 12 · pg_dump / restore round-trip · `standards-compat-2`

An administrator dumps with --no-unlogged-table-data to get a fast restore without rebuildable data. Restore still runs REFRESH MATERIALIZED VIEW for each unlogged matview, which can take hours on large views. The administrator finds out during the restore window.

Message #7 says the flag ignores matviews because they "are not tables", and calls this an open question. Users read the option as "skip data I marked as disposable", and an unlogged matview is exactly that. The documentation of the option and the refresh step in the dump need one consistent rule, which the thread has not agreed.

Thread: [#7 (Zsolt Parragi, 2026-09-24)](https://www.postgresql.org/message-id/CAN4CZFOLPsqmDoyU_BX8%3D%3DQx4_cvF5c4CZb-K3aUn2OESdDCjw%40mail.gmail.com)

**Question:** What should --no-unlogged-table-data do with unlogged matviews, for example dump them WITH NO DATA, and why is ignoring the flag the better contract?

Counter-check: held. Message #7 calls this an open question and nobody answered it. A dump with --no-unlogged-table-data still emits the refresh for an unlogged matview, so the restore cost is real.

### 4. An unlogged table refreshed from a plain view already avoids WAL

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, on-disk format / pg_upgrade, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user who needs a WAL-free cache today can keep the query in a regular view and fill an unlogged table from it. A refresh job runs `truncate cache; insert into cache select * from cache_v` in one transaction, or builds a new unlogged table and renames it into place. To get the same result built in, every tool that reads pg_class.relispopulated must change, as pgAdmin does in message #9, and pg_control gains a new field.

The thread does not compare the proposal with this existing idiom, which already gives the WAL saving in message #1 and needs no catalog or pg_control change. The proposal's real gains are refresh syntax and "unpopulated, not empty" after a crash. A marker row in the same unlogged table covers the second, because crash recovery removes the marker together with the data. The thread gives no refresh numbers, so a committer would likely ask whether these conveniences justify retyping a pg_class column and changing the pg_control layout.

Thread: [#1 (Zsolt Parragi, 2026-07-20)](https://www.postgresql.org/message-id/CAN4CZFO74UCnbu_nY-5vxju8p6_YKzUdOfhLVUuAGqwVMbCZHA%40mail.gmail.com), [#2 (Adam Brusselback, 2026-08-10)](https://www.postgresql.org/message-id/CAMjNa7fN6E4FwcFZpkDSr%3DXFD5o0y%3DrU68sZhYZbae7cz-3eFQ%40mail.gmail.com), [#9 (Jim Jones, 2026-10-13)](https://www.postgresql.org/message-id/bd706289-ce95-41bf-b417-595ec80168f8%40uni-muenster.de), [#10 (Zsolt Parragi, 2026-10-14)](https://www.postgresql.org/message-id/CAN4CZFO7NREnzNPkAmyeWRtMtAp6WPR1PqSp7wQUCb7yB--8Fg%40mail.gmail.com)

**Question:** What can an unlogged table filled from a regular view, with a marker row to detect a crash reset, not do that unlogged materialized views do?

Counter-check: held. The thread never compares the proposal with an unlogged table filled from a view, and gives no numbers. The marker row is weaker than the objection says, because unchecked queries read the table as empty, but that is a convenience gap.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Patch 0001 changes the pg_control layout and pg_resetwal, and patch 0002 changes the binary-upgrade path for matviews. |
| WAL / replication / standby | **yes** | The epoch is set in crash recovery, and the design defines behaviour on standby, promotion and PITR. |
| wire protocol | no |  |
| catalog changes | **yes** | pg_class.relispopulated (bool) is replaced by relpopulated (int8), and pg_matview_is_populated() is added to pg_proc. |
| ABI / extension API | **yes** | The rel.h populated macros are replaced by RelationIsPopulated() in matview.c, so extensions that use them must change. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds create unlogged materialized view and alter materialized view ... set { logged \| unlogged }, copied from the table syntax. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump reads relpopulated <> 0, and --no-unlogged-table-data does not apply to unlogged matviews. |
| psql UX conventions | **yes** | psql \d reports unlogged matviews, and tab completion gains the new syntax. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A new pg_control field and an epoch rule in recovery and pg_resetwal need care in later changes. |
| performance on the hot path | **yes** | The populated check now runs through RelationIsPopulated() on matview access; relcache.c and execUtils.c are touched. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | An unlogged table filled from a regular view already avoids WAL, and the thread does not compare the two. |

## 5. Questions the author must answer before anyone reviews code

1. During pg_upgrade, what does binary_upgrade_set_matview_populated() write for an unlogged matview whose old-cluster stamp is stale, and which test covers that case?
2. Why can relispopulated not stay as a bool that is exact for all existing relations, with the unlogged epoch kept in a separate column?
3. What should pg_dump --no-unlogged-table-data do with unlogged matviews, for example dump them WITH NO DATA?
4. How much WAL and refresh time does an unlogged matview save on a real workload, compared with an unlogged table filled from a regular view?
5. How does refresh materialized view concurrently work with unlogged storage?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows recovery, pg_control and pg_resetwal, and one willing to rule on the pg_class column change.
- **Liability:** The owner signs for a new pg_control field, the rule that every unlogged reset advances the epoch, and a changed pg_class column. All three last for every later release.
- **Author follow-up:** The author posted revisions up to v6 and addressed the reviewer's error message points, so the author is active in the thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Two users confirmed the use case, but the v6 series is about 83 KiB across recovery, pg_control, the catalog, pg_dump and psql. The pg_upgrade case and the catalog column shape need an answer, and a committer's view, before line-by-line review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Removing pg_class.relispopulated moves the cost to every catalog reader. Refuted (not specific): Duplicate of standards-compat-1, which makes the same point and adds the `relpopulated <> 0` hazard.
- `maintenance-fit-2` New pg_control invariant must be kept by every future unlogged-reset path. Refuted (no failure scenario): The scenario depends on a future patch that nobody has proposed. The objection names no existing path that discards unlogged storage without the LSN update, and 0001 covers crash recovery and pg_resetwal. The missing committer reply belongs to committability.
- `demand-1` Every client that reads pg_class.relispopulated breaks, for a few users' gain. Refuted (not specific): Duplicate of standards-compat-1. The cost to clients that read relispopulated is the same point, and that objection also names a design that avoids it.
- `demand-2` No numbers and no comparison with the unlogged-table workaround. Refuted (not specific): Duplicate of already-exists-1, which covers both the unlogged-table workaround and the missing refresh numbers.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected for a reason that still applies here: P9 closed because it sat idle, and P3 and P14 address other ideas. Message #1 reports that earlier threads stalled because pg_class cannot be updated during recovery, and the epoch design is aimed at that blocker.
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. The new syntax copies CREATE UNLOGGED TABLE and SET { LOGGED | UNLOGGED }, so it follows existing practice. The extra pg_controldata line is an addition and does not change existing lines.
- replication-ondisk: 1 objection(s). The LSN-epoch argument for crash, standby, promotion and PITR holds from the material. Each recovery sets an epoch above every stamp it replayed, and standbys do not read unlogged storage. One case could not be checked without the code: pg_resetwal on a standby that was shut down in recovery.
- security: 0 objection(s). The proposal has no new privilege or exposure aspect. Creating, altering and refreshing an unlogged matview use the existing matview ownership checks. pg_matview_is_populated() reports a state that pg_matviews and pg_class already show to all users. The proposal adds no new path for running code as another role, and it does not touch RLS, authentication or resource limits.
- maintenance-fit: 2 objection(s). Two objections raised. Not evaluated: hot-path cost of the new populated check, which depends on code not shown.
- already-exists: 1 objection(s). One objection from this angle. The auto-refresh worker and health check are deferred and could live in an extension, but they are not part of this proposal, so this review does not judge them.
- demand: 2 objection(s). Demand is plausible: the topic recurs, and the IVM and partial-refresh threads show that refresh cost hurts users. But it is not measured, and the cost of removing the pg_class column falls on everyone.

### Precedents retrieved

- P1: [Triggers on Materialized Views](https://commitfest.postgresql.org/patch/1948/) (cf, Returned with feedback, 2019-04-01) - judged not relevant
- P2: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Temporary materialized views](https://commitfest.postgresql.org/patch/1951/) (cf, Returned with feedback, 2019-03-15) - used in the review
- P4: [Triggers on materialized views](https://commitfest.postgresql.org/patch/1953/) (cf, Withdrawn, 2019-01-04) - judged not relevant
- P5: [EXPLAIN/EXPLAIN ANALYZE REFRESH MATERIALIZED VIEW](https://commitfest.postgresql.org/patch/2928/) (cf, Rejected, 2021-07-10) - judged not relevant; reason summarised from its thread
- P6: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P7: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - judged not relevant; reason summarised from its thread
- P8: [Allow granting of permission to REFRESH MATERIALIZED VIEW, CLUSTER, and REINDEX](https://commitfest.postgresql.org/patch/1605/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P9: [Detectable crashes and unlogged table resets](https://commitfest.postgresql.org/patch/3183/) (cf, Returned with feedback, 2022-03-07) - used in the review; reason summarised from its thread
- P10: [Unlogged relation copy is not fsync'd](https://commitfest.postgresql.org/patch/4517/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P11: [Add recovery to pg_control and remove backup_label](https://commitfest.postgresql.org/patch/4629/) (cf, Withdrawn, 2024-01-28) - judged not relevant; reason summarised from its thread
- P12: [\[PATCH\] Equivalence Class Filters](https://commitfest.postgresql.org/patch/3524/) (cf, Returned with feedback, 2022-12-17) - judged not relevant
- P13: [psql: add \si, \sm, \st and \sr functions to show CREATE commands for indexes, matviews, triggers and tables](https://commitfest.postgresql.org/patch/2664/) (cf, Returned with feedback, 2020-11-24) - judged not relevant
- P14: [Revert ill-considered change in pg_resetwal output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=918de37652138da7f7e54c79351178367d60f501) (revert, Reverted, 2022-09-15) - used in the review
- P15: [pg_restore cleanups](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=40b9c27014d90d77c61e8f5c77ddeb84fa9a6b69) (revert, Partially reverted, 2025-04-16) - judged not relevant
- P16: [rich constructor for plpy.Error class, possible raise structured exception](https://commitfest.postgresql.org/patch/387/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P17: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P18: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P19: [test pgbench time-related features](https://commitfest.postgresql.org/patch/3242/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P20: [Flush SLRU counters in checkpointer process](https://commitfest.postgresql.org/patch/4120/) (cf, Returned with feedback, 2023-09-04) - judged not relevant
- P21: [Fix hostaddr crash during non-blocking cancellation](https://commitfest.postgresql.org/patch/5741/) (cf, Withdrawn, 2025-07-07) - judged not relevant
- P22: [Revert recent SQL/JSON related commits](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4d7e24e0f4d05b546228488ccdc2848a80245ffb) (revert, Reverted, 2024-09-06) - judged not relevant
- P23: [Revert pg_relation_check_pages()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e152506adef4bc503ea7b8ebb4fedc0b8eebda81) (revert, Reverted, 2020-11-04) - judged not relevant
- P24: [Planning counters in pg_stat_statements](https://commitfest.postgresql.org/patch/1373/) (cf, Returned with feedback, 2018-01-25) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 158 s; cost reported by the CLI: $2.85
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:24:47.631Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
