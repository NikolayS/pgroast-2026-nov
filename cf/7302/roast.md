# Design pre-review: Optimize MCV stats for sortable types and utilize sorted-order properties

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7302](https://commitfest.postgresql.org/patch/7302/); [thread](https://www.postgresql.org/message-id/flat/tencent_0489DF3C961BD4D48D32F04F6D6EB4301308%40qq.com), 30 of 30 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The proposal fails at level 1b on maintenance-fit-1, already-exists-1 and demand-1. The author's release-build reruns show scalar = and < planning 5 to 6 percent slower than master, and the join gain was measured only with the hash threshold raised to 202. No workload in the thread shows planning time lost in MCV scans. v16 adds a sixth pg_statistic slot and duplicates MCV data for all users, so the benefit must be shown first.

Not checked: Only 3 of 30 messages and no code were read, so the level 2 objections on the sixth slot, the duplicated MCV data and collation version changes were not evaluated. The privilege and leakproof handling of the new paths could not be checked.

## Prior attempts

- [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://www.postgresql.org/message-id/flat/C71ABD3E-93C7-41A5-99AC-77279557D36A@Outlook.com) (related; Withdrawn, 2026-05-03). The visible messages do not show why. Ilia Evdokimov asked how often this narrow path runs, since most types use compute_scalar_stats(), and John Naylor questioned the threshold of 100. Both points were still open when the entry was withdrawn, on the day Ilia posted a cleanup review of v5 with no objection. (AI summary of the thread) Different now: P2 sped up MCV tracking inside compute_distinct_stats() for equality-only types with no catalog change. This proposal changes the stored MCV form for sortable types, adds a sixth statistics slot and changes planner consumers.

## Objections a reviewer is likely to raise

### 1. Permanent catalog change and duplicated MCV data for a gain not shown at defaults

Every installation stores two copies of each sortable column's MCV list and spends 2 to 3 percent more on every ANALYZE. Scalar = and < planning gets 5 to 6 percent slower in release builds. The only measured gain is on an eqjoinsel() path that the default hash threshold of 200 skips for 100+100 MCVs.

Message #29 retracts the debug-build speedups. It reports scalar regressions and join gains measured only at a hash threshold of 202, and calls the default-path benefit unproven. Message #7 argues the scan is cheap at normal targets, and no release data at large targets has answered that. Once STATISTIC_NUM_SLOTS is six and pg_stats has new columns, the project cannot take them back, so the costs fall on all users for a gain that is not yet demonstrated.

Precedent: [SIMD optimization for list_sort and tuple-sort](https://www.postgresql.org/message-id/flat/CO6PR11MB5620E3878444C023A7C8CA9C95222@CO6PR11MB5620.namprd11.prod.outlook.com) (Withdrawn, 2025-08-06)

**Question:** In a release build at default settings, which real workload spends enough planning time in MCV scans that v16 gives a measurable net gain over master?

<sub>fixable, level 1b, desirability; messages [#7](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com); `maintenance-fit-1`</sub>

### 2. Master's existing hash matching in eqjoinsel() already covers the measured join case

A user runs the default build with 100-entry MCV lists on both join sides, so master already takes the eqjoinsel() hash path. After upgrade they pay a sixth pg_statistic slot, duplicated MCV data and wider pg_stats, but planning time does not improve. Nobody sees this until someone benchmarks at the default threshold.

The only release-build gain shown is the join cases, which the author ran with the hash threshold raised to 202 to force the non-hash path (messages #29, #30). The author states the benefit over master's default hash path is unproven. The scalar = and < cases are 5 to 6 percent slower in release builds. So the gain found so far sits in a case master already handles with an existing mechanism, and the patch adds a second matching method for the same job.

**Question:** At the default hash threshold, which planning workloads become measurably faster with kind 8 than with master's existing eqjoinsel() hash path?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#7](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com); `already-exists-1`</sub>

### 3. No shown workload where MCV scans cost noticeable planning time

A committer merges a large statistics change to speed up planning. Users on default settings then see scalar = and < planning about 5 to 6 percent slower. Join gains appear only on a non-default path, so after release nobody can point to a workload that got faster.

The thread has no field report, -general or -bugs thread, or production story about planner time spent in MCV scans. The only other participant, in message #7, called the scan cheap at normal targets and the status quo better. The author's release reruns in messages #29 and #30 show scalar cases slower than master. The join gains used hash threshold 202, and the author says a benefit over the default hash path is unproven. The large earlier gains came from debug builds.

**Question:** Which real workloads, measured on release builds at default settings or at large statistics targets, spend a significant share of planning time in MCV comparisons that v16 removes?

<sub>fixable, level 1b, demand; messages [#7](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com); `demand-1`</sub>

## Questions for the author

1. In a release build at default settings, which workload plans measurably faster with v16 than with master, and by how much?
2. At the default hash threshold of 200, how does v16 join planning time compare with master's eqjoinsel() hash path?
3. What are the release-build planning times at large statistics targets, such as 10000 with long text under a nondeterministic collation?
4. Why is a sorted copy built from kind 1 when the planner loads the statistics, and cached, not enough?
5. How much do pg_statistic size, ANALYZE WAL volume and per-backend syscache memory grow against master at default and high statistics targets?

## Touch points

- **on-disk format / pg_upgrade:** v16 raises STATISTIC_NUM_SLOTS to six, which changes the pg_statistic row layout; upgraded clusters carry only kind 1 until the next ANALYZE.
- **WAL / replication / standby:** ANALYZE writes each sortable column's MCV values twice, so its WAL volume grows.
- **catalog changes:** New statistics kind 8, a sixth slot in pg_statistic, and new columns in pg_stats and pg_stats_ext_exprs.
- **ABI / extension API:** STATISTIC_NUM_SLOTS and the VacAttrStats slot arrays change size, and a new get_attstatsslot_mcv() is added.
- **planner behaviour change:** var_eq_const(), mcv_selectivity(), eqjoinsel() and the range helpers get new lookup paths.
- **pg_dump / restore round-trip:** pg_dump statistics export and pg_restore_attribute_stats() must carry and validate a kind 1 and kind 8 pair.
- **security / privileges:** New pg_stats columns must keep the existing privilege filter, and ordered lookups call the "<" operator on user data.
- **maintenance burden ("who maintains this in 10 years"):** Two stored MCV forms, pair validation in import, and about 3,400 new lines across 27 files.
- **performance on the hot path:** Selectivity estimation runs for every planned query; release builds show scalar cases 5 to 6 percent slower.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Master's eqjoinsel() hash path already handles the join case at the default threshold of 200.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the selectivity functions in selfuncs.c, ANALYZE and the statistics import and export code. None has spoken in the visible thread.
- Not now, show demand first: Release-build planning-time measurements at default settings and at large statistics targets that show a net gain over master, including the default eqjoinsel() hash path. A real workload where MCV scans take a visible share of planning time would also help, together with measured catalog, WAL and syscache growth.
- Priority: needs design rethink; audience niche; review cost high. The stated target is large statistics targets with expensive comparisons, and release builds show no gain at defaults. v16 changes the catalog, views, import, pg_dump and postgres_fdw in 27 files, and the thread has not discussed a read-time sorted copy that would avoid the catalog change.

---

[roast-full.md](roast-full.md) has 4 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
