# Design pre-review: Unlogged materialized views

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7167](https://commitfest.postgresql.org/patch/7167/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFO74UCnbu_nY-5vxju8p6_YKzUdOfhLVUuAGqwVMbCZHA%40mail.gmail.com), 12 of 12 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support in the thread, and the LSN epoch answers the old blocker of catalog writes during recovery. Three design points are still open. replication-ondisk-1 shows a path where pg_upgrade can mark an empty unlogged matview as populated. standards-compat-1 shows that removing relispopulated breaks clients on clusters that never use the feature. standards-compat-2 is a pg_dump rule the thread left open.

Not checked: Without the code, this review could not check what binary upgrade writes for a stale stamp, or how pg_resetwal behaves on a standby shut down in recovery. It also could not check the cost of the populated check on matview access.

## Prior attempts

- [Detectable crashes and unlogged table resets](https://www.postgresql.org/message-id/flat/62750df5b126e1d8ee039a79ef3cc64ac3d47cd5.camel@j-davis.com) (similar idea; Returned with feedback, 2022-03-07). "This is too long, so let's discard this patch for now." ([message](https://www.postgresql.org/message-id/YiM8fhx2udJJwlJ7%40paquier.xyz)) Different now: P9 recorded the last crash recovery in pg_control so applications could detect unlogged resets, with no consumer in core. This proposal stores a similar LSN but uses it inside core to invalidate unlogged matviews, and it also covers promotion and PITR.

## Objections a reviewer is likely to raise

### 1. pg_upgrade may mark a crash-emptied unlogged matview as populated

A PG20 cluster crashes and recovery resets an unlogged matview to its empty init fork, but nobody refreshes it. The user then runs pg_upgrade to PG21, and the new cluster marks the matview populated while the transferred file is empty. Queries return no rows without any error, and the user may only notice from wrong reports.

Patch 0002 says pg_dump reads `relpopulated <> 0` and, in binary upgrade, calls binary_upgrade_set_matview_populated(). A stale epoch stamp is non-zero, so the dump treats it as populated. The old stamp is an LSN from the old cluster's WAL, so it means nothing in the new cluster, whose epoch starts at the initdb bootstrap checkpoint. Copying the stamp would invalidate every unlogged matview on upgrade, and re-stamping would validate empty ones. The pg_upgrade claim in the brief covers only the pg_resetwal step and does not address this case.

**Question:** During pg_upgrade, what does binary_upgrade_set_matview_populated() write for an unlogged matview whose old-cluster stamp is stale, and which test covers that case?

<sub>fixable, level 2, on-disk format / pg_upgrade; messages [#7](https://www.postgresql.org/message-id/CAN4CZFOLPsqmDoyU_BX8%3D%3DQx4_cvF5c4CZb-K3aUn2OESdDCjw%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAN4CZFOrHgLMBngZE21CsfC_Mqj4vbeRdkZouUW6dNb1mu2f3Q%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Removing pg_class.relispopulated breaks every client for a narrow feature

A user upgrades to the new release and opens pgAdmin or a monitoring script that reads c.relispopulated. The query fails with "column c.relispopulated does not exist", even though the cluster has no unlogged matview. The user finds out at the first catalog query after the upgrade.

Message #9 shows pgAdmin failing, and message #11 notes it is not the only tool affected. The author says in message #10 that the column cannot stay because it would go stale. That is true only for unlogged matviews. A bool column could stay correct for every existing relation, with the epoch held in a separate column, so the cost would fall only on users of the new feature. The new int8 column also mixes a flag with an LSN typed as int8, and a natural rewrite such as relpopulated <> 0 reports a stale unlogged matview as populated.

**Question:** Why can't relispopulated stay as a bool that is exact for all existing relations, with the unlogged epoch kept in a separate column or in pg_matview_is_populated() only?

<sub>fixable, level 2, catalog changes; messages [#9](https://www.postgresql.org/message-id/bd706289-ce95-41bf-b417-595ec80168f8%40uni-muenster.de), [#10](https://www.postgresql.org/message-id/CAN4CZFO7NREnzNPkAmyeWRtMtAp6WPR1PqSp7wQUCb7yB--8Fg%40mail.gmail.com), [#11](https://www.postgresql.org/message-id/999c81c2-77ca-4903-883b-e01a31d12aab%40uni-muenster.de); `standards-compat-1`</sub>

### 3. pg_dump --no-unlogged-table-data skips unlogged tables but not unlogged matviews

An administrator dumps with --no-unlogged-table-data to get a fast restore without rebuildable data. Restore still runs REFRESH MATERIALIZED VIEW for each unlogged matview, which can take hours on large views. The administrator finds out during the restore window.

Message #7 says the flag ignores matviews because they "are not tables", and calls this an open question. Users read the option as "skip data I marked as disposable", and an unlogged matview is exactly that. The documentation of the option and the refresh step in the dump need one consistent rule, which the thread has not agreed.

**Question:** What should --no-unlogged-table-data do with unlogged matviews, for example dump them WITH NO DATA, and why is ignoring the flag the better contract?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#7](https://www.postgresql.org/message-id/CAN4CZFOLPsqmDoyU_BX8%3D%3DQx4_cvF5c4CZb-K3aUn2OESdDCjw%40mail.gmail.com); `standards-compat-2`</sub>

### 4. An unlogged table refreshed from a plain view already avoids WAL

A user who needs a WAL-free cache today can keep the query in a regular view and fill an unlogged table from it. A refresh job runs `truncate cache; insert into cache select * from cache_v` in one transaction, or builds a new unlogged table and renames it into place. To get the same result built in, every tool that reads pg_class.relispopulated must change, as pgAdmin does in message #9, and pg_control gains a new field.

The thread does not compare the proposal with this existing idiom, which already gives the WAL saving in message #1 and needs no catalog or pg_control change. The proposal's real gains are refresh syntax and "unpopulated, not empty" after a crash. A marker row in the same unlogged table covers the second, because crash recovery removes the marker together with the data. The thread gives no refresh numbers, so a committer would likely ask whether these conveniences justify retyping a pg_class column and changing the pg_control layout.

**Question:** What can an unlogged table filled from a regular view, with a marker row to detect a crash reset, not do that unlogged materialized views do?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAN4CZFO74UCnbu_nY-5vxju8p6_YKzUdOfhLVUuAGqwVMbCZHA%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAMjNa7fN6E4FwcFZpkDSr%3DXFD5o0y%3DrU68sZhYZbae7cz-3eFQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/bd706289-ce95-41bf-b417-595ec80168f8%40uni-muenster.de), [#10](https://www.postgresql.org/message-id/CAN4CZFO7NREnzNPkAmyeWRtMtAp6WPR1PqSp7wQUCb7yB--8Fg%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. During pg_upgrade, what does binary_upgrade_set_matview_populated() write for an unlogged matview whose old-cluster stamp is stale, and which test covers that case?
2. Why can relispopulated not stay as a bool that is exact for all existing relations, with the unlogged epoch kept in a separate column?
3. What should pg_dump --no-unlogged-table-data do with unlogged matviews, for example dump them WITH NO DATA?
4. How much WAL and refresh time does an unlogged matview save on a real workload, compared with an unlogged table filled from a regular view?
5. How does refresh materialized view concurrently work with unlogged storage?

## Touch points

- **on-disk format / pg_upgrade:** Patch 0001 changes the pg_control layout and pg_resetwal, and patch 0002 changes the binary-upgrade path for matviews.
- **WAL / replication / standby:** The epoch is set in crash recovery, and the design defines behaviour on standby, promotion and PITR.
- **catalog changes:** pg_class.relispopulated (bool) is replaced by relpopulated (int8), and pg_matview_is_populated() is added to pg_proc.
- **ABI / extension API:** The rel.h populated macros are replaced by RelationIsPopulated() in matview.c, so extensions that use them must change.
- **SQL-standard conflict or non-standard syntax:** The patch adds create unlogged materialized view and alter materialized view ... set { logged | unlogged }, copied from the table syntax.
- **pg_dump / restore round-trip:** pg_dump reads relpopulated <> 0, and --no-unlogged-table-data does not apply to unlogged matviews.
- **psql UX conventions:** psql \d reports unlogged matviews, and tab completion gains the new syntax.
- **maintenance burden ("who maintains this in 10 years"):** A new pg_control field and an epoch rule in recovery and pg_resetwal need care in later changes.
- **performance on the hot path:** The populated check now runs through RelationIsPopulated() on matview access; relcache.c and execUtils.c are touched.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** An unlogged table filled from a regular view already avoids WAL, and the thread does not compare the two.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows recovery, pg_control and pg_resetwal, and one willing to rule on the pg_class column change.
- Priority: fix design first; audience a sizeable minority; review cost high. Two users confirmed the use case, but the v6 series is about 83 KiB across recovery, pg_control, the catalog, pg_dump and psql. The pg_upgrade case and the catalog column shape need an answer, and a committer's view, before line-by-line review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
