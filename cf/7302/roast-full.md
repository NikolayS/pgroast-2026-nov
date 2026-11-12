# Design pre-review: Optimize MCV stats for sortable types and utilize sorted-order properties

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7302](https://commitfest.postgresql.org/patch/7302/) (Needs review); authors: Zizhuan Liu (liuzizhuan)
- Thread: [Optimize MCV stats for sortable types and utilize sorted-order properties](https://www.postgresql.org/message-id/flat/tencent_0489DF3C961BD4D48D32F04F6D6EB4301308%40qq.com), started 2026-09-15
- Messages read: 30 of 30
- Patch prose read: v15-0001-Optimize-MCV-statistics-for-sortable-types.patch, v1-0001-Optimize-MCV-stats-for-sortable-types-and-utilize.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The proposal fails at level 1b on maintenance-fit-1, already-exists-1 and demand-1. The author's release-build reruns show scalar = and < planning 5 to 6 percent slower than master, and the join gain was measured only with the hash threshold raised to 202. No workload in the thread shows planning time lost in MCV scans. v16 adds a sixth pg_statistic slot and duplicates MCV data for all users, so the benefit must be shown first.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 30 messages and no code were read, so the level 2 objections on the sixth slot, the duplicated MCV data and collation version changes were not evaluated. The privilege and leakproof handling of the new paths could not be checked.

## The proposal, restated

**Problem.** The planner scans the frequency-ordered MCV list linearly, calling the comparison function for each entry, when it estimates equality, inequality and join selectivity. The author wants to cut this planner CPU cost, mainly for large statistics targets and expensive comparisons such as long text with a nondeterministic collation.

**Proposed design.** ANALYZE stores a new statistics kind, STATISTIC_KIND_MCV_VALUE_SORTED (8), that holds the MCV pairs in ascending value order. The planner uses endpoint checks, binary search and merge matching in var_eq_const(), mcv_selectivity(), eqjoinsel() and the range helpers. Up to v15, kind 8 replaced kind 1 for sortable types; v16 keeps both kinds and raises STATISTIC_NUM_SLOTS from five to six.

**User-visible surface.**
- New statistics kind 8, STATISTIC_KIND_MCV_VALUE_SORTED, in pg_statistic; extensions reading MCV slots see it.
- v16: STATISTIC_NUM_SLOTS goes from 5 to 6, adding stakind6/staop6/stacoll6/stanumbers6/stavalues6-style catalog columns.
- v16: new most_common_vals_value_sorted and most_common_freqs_value_sorted columns in pg_stats and pg_stats_ext_exprs; v15 instead reordered the existing columns.
- Optional most_common_vals_kind parameter for pg_restore_attribute_stats() and the expr arguments of pg_restore_extended_stats() (v8 to v15); v16 imports a kind 1 and kind 8 pair.
- New C function get_attstatsslot_mcv() for explicit MCV representation selection; pg_dump and postgres_fdw statistics export change.

**What the author claims.**
- ANALYZE overhead is about 2 to 3 percent in the author's tests.
- Release-build reruns show join planning time about 14 to 16 percent faster with v16, using a hash threshold of 202.
- Release builds show scalar = and < cases about 5 to 6 percent slower than master; earlier 40 to 60 percent gains came from debug builds.
- Plans and row estimates match master in the tested cases; v15 had a join estimate defect that v16 fixes.
- v16 touches 27 files, with about 3,400 inserted lines.

**Thread state.** Only Ilia Evdokimov replied, arguing the MCV scan is cheap at normal targets, a new statistics kind wastes slots, and the status quo is better. The author has posted 16 versions alone and now asks whether the paired design with a sixth slot is acceptable.

## 2. Prior attempts

### ANALYZE: hash-accelerate MCV tracking for equality-only types (related)

- **Link:** [https://commitfest.postgresql.org/patch/6422/](https://commitfest.postgresql.org/patch/6422/)
- **Thread:** [https://www.postgresql.org/message-id/flat/C71ABD3E-93C7-41A5-99AC-77279557D36A@Outlook.com](https://www.postgresql.org/message-id/flat/C71ABD3E-93C7-41A5-99AC-77279557D36A@Outlook.com)
- **Date:** decided 2026-05-03; started 2026-01-21
- **Outcome:** Withdrawn (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. Ilia Evdokimov asked how often this narrow path runs, since most types use compute_scalar_stats(), and John Naylor questioned the threshold of 100. Both points were still open when the entry was withdrawn, on the day Ilia posted a cleanup review of v5 with no objection.
- **Reason (quote):** quote unavailable
- **What is different this time:** P2 sped up MCV tracking inside compute_distinct_stats() for equality-only types with no catalog change. This proposal changes the stored MCV form for sortable types, adds a sixth statistics slot and changes planner consumers.


## 3. Objections a reviewer is likely to raise

### 1. Permanent catalog change and duplicated MCV data for a gain not shown at defaults

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 4 · desirability, catalog changes, performance on the hot path, maintenance burden ("who maintains this in 10 years"), on-disk format / pg_upgrade · `maintenance-fit-1`

Every installation stores two copies of each sortable column's MCV list and spends 2 to 3 percent more on every ANALYZE. Scalar = and < planning gets 5 to 6 percent slower in release builds. The only measured gain is on an eqjoinsel() path that the default hash threshold of 200 skips for 100+100 MCVs.

Message #29 retracts the debug-build speedups. It reports scalar regressions and join gains measured only at a hash threshold of 202, and calls the default-path benefit unproven. Message #7 argues the scan is cheap at normal targets, and no release data at large targets has answered that. Once STATISTIC_NUM_SLOTS is six and pg_stats has new columns, the project cannot take them back, so the costs fall on all users for a gain that is not yet demonstrated.

**Precedents.**
- [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (Withdrawn, 2025-08-06); thread: [#1](https://www.postgresql.org/message-id/flat/CO6PR11MB5620E3878444C023A7C8CA9C95222@CO6PR11MB5620.namprd11.prod.outlook.com), [#2](https://www.postgresql.org/message-id/flat/CO1PR11MB506011C3052AE779750D9789EDA02@CO1PR11MB5060.namprd11.prod.outlook.com)

Thread: [#7 (Ilia Evdokimov, 2026-09-21)](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com)

**Question:** In a release build at default settings, which real workload spends enough planning time in MCV scans that v16 gives a measurable net gain over master?

Counter-check: held. Messages #29 and #30 concede each point: scalar cases are slower, join gains are off the default path, and the costs are not yet justified. P4 concerns an external library and must not be used as support; new measurements could answer this, so fixable.

### 2. Master's existing hash matching in eqjoinsel() already covers the measured join case

fixable (first rated fatal; changed in the counter-check) · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user runs the default build with 100-entry MCV lists on both join sides, so master already takes the eqjoinsel() hash path. After upgrade they pay a sixth pg_statistic slot, duplicated MCV data and wider pg_stats, but planning time does not improve. Nobody sees this until someone benchmarks at the default threshold.

The only release-build gain shown is the join cases, which the author ran with the hash threshold raised to 202 to force the non-hash path (messages #29, #30). The author states the benefit over master's default hash path is unproven. The scalar = and < cases are 5 to 6 percent slower in release builds. So the gain found so far sits in a case master already handles with an existing mechanism, and the patch adds a second matching method for the same job.

Thread: [#7 (Ilia Evdokimov, 2026-09-21)](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com)

**Question:** At the default hash threshold, which planning workloads become measurably faster with kind 8 than with master's existing eqjoinsel() hash path?

Counter-check: held. Message #29 says the join gain was measured with the hash threshold raised to 202 and is unproven against master's default hash path. A default-threshold or large-target measurement could answer this, and the hash path does not cover the scalar cases, so fixable.

### 3. No shown workload where MCV scans cost noticeable planning time

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer merges a large statistics change to speed up planning. Users on default settings then see scalar = and < planning about 5 to 6 percent slower. Join gains appear only on a non-default path, so after release nobody can point to a workload that got faster.

The thread has no field report, -general or -bugs thread, or production story about planner time spent in MCV scans. The only other participant, in message #7, called the scan cheap at normal targets and the status quo better. The author's release reruns in messages #29 and #30 show scalar cases slower than master. The join gains used hash threshold 202, and the author says a benefit over the default hash path is unproven. The large earlier gains came from debug builds.

Thread: [#7 (Ilia Evdokimov, 2026-09-21)](https://www.postgresql.org/message-id/87fde6ac-3647-4fce-be92-af27f6c132fa%40tantorlabs.com), [#29 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_5E363FDABF09F7EF03CB7A6AEEC60D78A307%40qq.com), [#30 (ZizhuanLiu X-MAN, 2026-11-09)](https://www.postgresql.org/message-id/tencent_749BDFF289FAAE51085EA06D8B967AE2F905%40qq.com)

**Question:** Which real workloads, measured on release builds at default settings or at large statistics targets, spend a significant share of planning time in MCV comparisons that v16 removes?

Counter-check: held. The thread has no field report or production workload, and the only other participant argued for the status quo in message #7. The "unclear" list in the brief confirms that no release measurement at large statistics targets exists.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | v16 raises STATISTIC_NUM_SLOTS to six, which changes the pg_statistic row layout; upgraded clusters carry only kind 1 until the next ANALYZE. |
| WAL / replication / standby | **yes** | ANALYZE writes each sortable column's MCV values twice, so its WAL volume grows. |
| wire protocol | no |  |
| catalog changes | **yes** | New statistics kind 8, a sixth slot in pg_statistic, and new columns in pg_stats and pg_stats_ext_exprs. |
| ABI / extension API | **yes** | STATISTIC_NUM_SLOTS and the VacAttrStats slot arrays change size, and a new get_attstatsslot_mcv() is added. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | var_eq_const(), mcv_selectivity(), eqjoinsel() and the range helpers get new lookup paths. |
| pg_dump / restore round-trip | **yes** | pg_dump statistics export and pg_restore_attribute_stats() must carry and validate a kind 1 and kind 8 pair. |
| psql UX conventions | no |  |
| security / privileges | **yes** | New pg_stats columns must keep the existing privilege filter, and ordered lookups call the "<" operator on user data. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Two stored MCV forms, pair validation in import, and about 3,400 new lines across 27 files. |
| performance on the hot path | **yes** | Selectivity estimation runs for every planned query; release builds show scalar cases 5 to 6 percent slower. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Master's eqjoinsel() hash path already handles the join case at the default threshold of 200. |

## 5. Questions the author must answer before anyone reviews code

1. In a release build at default settings, which workload plans measurably faster with v16 than with master, and by how much?
2. At the default hash threshold of 200, how does v16 join planning time compare with master's eqjoinsel() hash path?
3. What are the release-build planning times at large statistics targets, such as 10000 with long text under a nondeterministic collation?
4. Why is a sorted copy built from kind 1 when the planner loads the statistics, and cached, not enough?
5. How much do pg_statistic size, ANALYZE WAL volume and per-backend syscache memory grow against master at default and high statistics targets?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the selectivity functions in selfuncs.c, ANALYZE and the statistics import and export code. None has spoken in the visible thread.
- **Liability:** The owner signs for a six-slot pg_statistic layout, statistics kind 8, new pg_stats columns and the pair rules in statistics import. These cannot be removed after release.
- **Author follow-up:** Strong sign: the author posted 16 versions in two months and reported the release-build regressions without being asked.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Release-build planning-time measurements at default settings and at large statistics targets that show a net gain over master, including the default eqjoinsel() hash path. A real workload where MCV scans take a visible share of planning time would also help, together with measured catalog, WAL and syscache growth.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The stated target is large statistics targets with expensive comparisons, and release builds show no gain at defaults. v16 changes the catalog, views, import, pg_dump and postgres_fdw in 27 files, and the thread has not discussed a read-time sorted copy that would avoid the catalog change.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) Stored value order goes stale when the collation sort order changes. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Sixth statistics slot changes pg_statistic layout and VacAttrStats for everyone. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-2` (fixable, level 2) Duplicate MCV payload grows pg_statistic, syscache memory and ANALYZE WAL. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-3` (fixable, level 2) Sorted order could be derived at read time without a catalog change. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Derived planner data becomes a catalog, view and import contract. Refuted (not specific): Duplicate of maintenance-fit-3, which asks the same read-time question. The failure scenario here is also weak: message #30 says import accepts kind 1 alone, and the planner then uses kind 1 as master does.
- `standards-compat-2` A sixth slot breaks SQL that enumerates stakind1 to stakind5. Refuted (not specific): Duplicate of replication-ondisk-1, which covers the same slot 6 scenario and adds the VacAttrStats consumers.
- `maintenance-fit-2` Two stored MCV forms must stay in sync across every producer. Refuted (already handled): Message #30 says import rejects kind 8 alone and validates pairs, and message #9 shows ANALYZE replaces the whole row. A producer that writes only kind 1 leaves the planner on master's kind 1 path, and get_attstatsslot_mcv() makes the consumer's choice explicit.
- `demand-2` Every installation pays for a gain few workloads would see. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about costs for all users against a narrow gain, with more detail from the release measurements.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent records a rejection reason that applies to value-sorted MCVs or to raising STATISTIC_NUM_SLOTS. P4 shares only the 'performance patch' theme. Message #1 builds on an earlier thread and CF entry, and message #28 cites three older discussions in support of the paired design, but the material does not say how those ended, so no prior-rejection objection can rest on them.
- standards-compat: 3 objection(s). Objections were raised; see the list.
- replication-ondisk: 2 objection(s). Physical replication, hot standby and the wire protocol are not affected. pg_upgrade carries only kind 1, and the planner falls back to it until the next ANALYZE, which looks harmless.
- security: 0 objection(s). The proposal reorders and duplicates data that pg_stats and pg_stats_ext_exprs already expose, so it gives no role access to anything new. Message #26 drops the pg_statistic join that failed for non-superusers and keeps the views' existing filters. Message #30 makes the ordered paths depend on permission to call the actual comparator, and has import reject unsorted or kind-8-only input. Without the code I could not check four things: that the v16 view columns use the same privilege and row-security filter, that the leakproof check now covers the '<' operator used for equality lookups, that ordering validation on restore cannot be bypassed, and that the sixth slot adds no DoS path.
- maintenance-fit: 3 objection(s). Objections were found, so this note does not apply.
- already-exists: 1 objection(s). No contrib module or extension does value-ordered MCV lookup. Histogram bounds exclude MCVs, so they do not replace the scalar fast paths. The thread settled this point (message #30).
- demand: 2 objection(s). Two objections apply. Both are 'not now, show demand' at level 1b. Release-build measurements of real workloads could change the answer.

### Precedents retrieved

- P1: [Add support for (Var op Var) clause in extended MCV statistics](https://commitfest.postgresql.org/patch/5180/) (cf, Returned with feedback, 2024-09-30) - judged not relevant; reason summarised from its thread
- P2: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - used in the review; reason summarised from its thread
- P3: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant; reason summarised from its thread
- P4: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - used in the review; reason summarised from its thread
- P5: [Provide the facility to set binary format output for specific OID's per session](https://commitfest.postgresql.org/patch/3777/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P6: [Binary I/O for isn extension](https://commitfest.postgresql.org/patch/639/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P7: [Add uri percent-encoding for binary data](https://commitfest.postgresql.org/patch/2299/) (cf, Returned with feedback, 2020-07-29) - judged not relevant; reason summarised from its thread
- P8: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P9: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P10: [Revert "Common SQL/JSON clauses"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1460fc5942591fdb6bee0bc8342019ede31ff3b6) (revert, Reverted, 2022-03-22) - judged not relevant
- P11: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - judged not relevant
- P12: [Using multicolumn statistics for more precise join selectivity estimatation](https://commitfest.postgresql.org/patch/2386/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [Invalid permission check in pg_stats for functional indexes](https://commitfest.postgresql.org/patch/2274/) (cf, Returned with feedback, 2020-07-05) - judged not relevant
- P14: [Improve management of SLRU statistics collection.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=81ca8686305c4c62d723ab224ad5c414f350a3a0) (revert, Partially reverted, 2020-05-13) - judged not relevant
- P15: [Improving the heapgetpage function improves performance in common scenarios](https://commitfest.postgresql.org/patch/4524/) (cf, Withdrawn, 2023-09-06) - judged not relevant
- P16: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant
- P17: [calculate and use most common type for polymorphic arguments](https://commitfest.postgresql.org/patch/258/) (cf, Rejected, 2015-07-22) - judged not relevant
- P18: [Improve join selectivity estimation using extended statistics](https://commitfest.postgresql.org/patch/2959/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P19: [Rename constants in preparation of variableness](https://commitfest.postgresql.org/patch/4828/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P20: [pg_stat_statements: add last_execution_start column](https://commitfest.postgresql.org/patch/6624/) (cf, Withdrawn, 2026-08-21) - judged not relevant
- P21: [\[WIP\] Show a human-readable n_distinct in pg_stats view](https://commitfest.postgresql.org/patch/2061/) (cf, Withdrawn, 2019-08-28) - judged not relevant
- P22: [Full merge join on comparison clause](https://commitfest.postgresql.org/patch/1141/) (cf, Rejected, 2018-11-21) - judged not relevant
- P23: [Patch to implement missing join selectivity estimation for range types](https://commitfest.postgresql.org/patch/3821/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P24: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 186 s; cost reported by the CLI: $5.30
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 109 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:38:19.838Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
