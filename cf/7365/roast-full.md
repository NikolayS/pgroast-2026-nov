# Design pre-review: ATTACH PARTITION cost grows linearly with pg_constraint size (seqscan in CloneFkReferenced), much worse since not-null constraints are in pg_constraint (PG 18)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7365](https://commitfest.postgresql.org/patch/7365/) (Needs review); authors: Manuel Reyes Bravo (manu15)
- Thread: [ATTACH PARTITION cost grows linearly with pg_constraint size (seqscan in CloneFkReferenced), much worse since not-null constraints are in pg_constraint (PG 18)](https://www.postgresql.org/message-id/flat/trinity-08d3329b-f6b2-4c11-91d7-2198e50bf4fc-1790681922354%40trinity-msg-rest-gmx-gmx-live-58cc8f554d-c56lm), started 2026-09-29
- Messages read: 12 of 12
- Patch prose read: none (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection stops the proposal at levels 1a, 1b or 2. The problem is stated with a reproducer, demand is shown by a production report, and the design copies the existing contypid index. The one surviving objection, maintenance-fit-1, is a measurement request about index maintenance cost on bulk DDL. A committer who knows the catalog code must weigh that cost and sign the catalog change.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v4 patch and the Tom Lane message that shaped it are not in the visible material. The four converted lookups, the index name and the contype check in each loop could not be checked.

## The proposal, restated

**Problem.** ATTACH PARTITION and CREATE TABLE ... PARTITION OF scan all of pg_constraint to find foreign keys that reference the parent, because no index covers confrelid. Since not-null constraints moved into pg_constraint in PG 18, the catalog is much larger in schemas with many partitions. The cost per ATTACH now grows with catalog size, even when no foreign key exists.

**Proposed design.** The patch adds a plain non-unique catalog index on pg_constraint(confrelid), like the existing one on contypid. Lookups that filter on confrelid pass this index to systable_beginscan and check contype in the loop. v4 covers CloneFkReferenced, GetParentedForeignKeyRefs, ATPrepChangePersistence and heap_truncate_find_FKs.

**User-visible surface.**
- New system index on pg_constraint (confrelid); its name is not in the visible material.
- Catalog version bump, left to the committer.

**What the author claims.**
- With about 1M constraints, ATTACH, DETACH, PARTITION OF, TRUNCATE and SET UNLOGGED drop from about 50-90 ms to under 2 ms, about 80 times faster.
- A commit loop over an ON COMMIT DELETE ROWS table with a foreign key drops from about 3.5 s to a few ms.
- Creating not-null or foreign-key constraints costs the same as before.
- The index is about 6 MiB next to a 340 MiB catalog; an earlier version measured about 5% extra on bulk DDL.
- Existing regression tests already exercise all call sites, so no new tests are needed.

**Thread state.** Álvaro Herrera agreed with the approach and asked for the CommitFest entry, but deferred review because PG 19 and older bugs come first. v4 and Tom Lane's input are in a separate thread that is not shown.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Unreconciled cost of the full index on every constraint insert

fixable · level 2, rubric item 4 · performance on the hot path, catalog changes · `maintenance-fit-1`

A user restores a large schema with pg_restore, or runs a migration that creates many tables with NOT NULL columns. Each not-null row in pg_constraint now also inserts an index entry with confrelid = 0. Users without partitions or foreign keys pay this cost and gain nothing.

Message #4 measured about 5% extra on bulk DDL for v1. Message #11 says constraint creation costs the same, but shows no run that reconciles the two numbers. Message #7 notes that about 90% of the index entries are not used, so nearly all of the maintenance cost lands on DDL that never uses the index. The shape copies the existing contypid index and has support in the thread, so this is a measurement gap, not a design flaw.

Thread: [#4 (Manu, 2026-09-29)](https://www.postgresql.org/message-id/179069810491.316007.4145107668471017354%40gmail.com), [#7 (Álvaro Herrera, 2026-09-29)](https://www.postgresql.org/message-id/arv-UHG6c-raryI-%40alvherre.pgsql), [#11 (Manu, 2026-10-16)](https://www.postgresql.org/message-id/179211577746.206608.6783271124892214599%40gmail.com)

**Question:** What is the v4 overhead on a schema-only pg_restore with about 1M not-null constraints, and why did the 5% seen in v1 go away?

Counter-check: held. Message #4 reports about 5% extra on bulk DDL and message #11 says constraint creation costs the same, with no run shown that explains the change. The thread weighed the cost and Álvaro accepted the approach, so this stays a measurement request.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | A new system index on pg_constraint changes the initial catalog contents and needs a catalog version bump. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds a non-unique index on pg_constraint(confrelid). |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The slowdown is a PG 18 regression, but a new catalog index cannot go into released branches; the entry targets version 20. |
| maintenance burden ("who maintains this in 10 years") | **yes** | One more catalog index to maintain, and four lookup sites now depend on it. |
| performance on the hot path | **yes** | Every pg_constraint insert, including each not-null row, also inserts an index entry with confrelid = 0. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What is the v4 overhead on a schema-only pg_restore with about 1M not-null constraints, and why did the 5% seen in v1 go away?
2. How do the pg_depend lookup and the trigger-based early exit from message #1 compare with the index on the same reproducer?
3. Do pg_upgrade or tools that list catalog indexes need any change for the new index?
4. Is any fix planned for PG 18 and 19, where the catalog index cannot be added?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera, who takes part in the thread, agreed with the approach but deferred review until PG 19 work and older bugs are done. The owner needs to know pg_constraint and the partition foreign-key code.
- **Liability:** The owner signs for a permanent catalog index, its cost on every constraint insert, and the correctness of four converted lookups in ATTACH, DETACH, TRUNCATE and SET UNLOGGED.
- **Author follow-up:** The author reproduced the report, offered to test alternatives, and maintains the CommitFest entry; the brief describes a v4, so the patch has been revised several times.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Message #1 shows a production regression since PG 18 that affects schemas with many partitions, and the fix copies the existing contypid index. No patch text was read; review cost is judged from the brief's description of one index and four call sites.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent matches. [P16] is about memory use when creating foreign keys between partitioned tables, a different idea, and it has no recorded reason. Message #3 mentions an earlier look at this index, which stalled because catalogs do not support partial indexes. v4 uses a plain non-unique index, as in message #11, so that blocker no longer applies.
- standards-compat: 0 objection(s). From a standards and compatibility angle the user-facing contract does not change. There is no syntax, keyword, GUC, psql change or new object type, and pg_dump does not emit system catalog indexes. pg_upgrade rebuilds catalogs through initdb, and the catversion bump keeps the change master-only, which message #5 asked for. The only visible change is one more row in pg_indexes and pg_class for pg_constraint, which does not change query results.
- replication-ondisk: 0 objection(s). The only persistent change is one plain non-unique btree on pg_constraint(confrelid), like the existing contypid index. It needs no new WAL record, no change to the user data format and no pg_upgrade step, because initdb builds the new cluster's catalogs. Backpatching was dropped in message #5, so there is no ABI concern. The extra WAL from catalog DDL comes with a 5% bulk DDL figure from v1 and a later claim of no cost. That conflict belongs to the performance angle and has no failure scenario on replication or on-disk state.
- security: 0 objection(s). No privilege or exposure aspect. The patch adds a plain catalog index on pg_constraint(confrelid) and points existing internal lookups at it; it adds no new command, function, privilege or view, and catalog contents stay visible exactly as before. Index growth follows constraint creation, which already needs table ownership, so it opens no new denial-of-service path.
- maintenance-fit: 1 objection(s). Beyond the overhead question, the fit is good. The index copies the existing contypid index, is master-only and easy to replace once partial catalog indexes exist, adds no API or external dependency, and has support from a committer who knows the area (messages #5 and #7).
- already-exists: 0 objection(s). No existing feature, setting or extension gives this result. The seqscan sits inside CloneFkReferenced, GetParentedForeignKeyRefs, ATPrepChangePersistence and heap_truncate_find_FKs, so only a core change can help. The one existing mechanism, pg_depend_reference_index (idea (a) in message #1), was not measured, but a plain index copies the contypid precedent, and message #11 reports that Tom Lane endorsed it.
- demand: 0 objection(s). Demand is shown. Message #1 is a production report with a reproducer and run times growing from 1:03 to 6:16. Message #2 confirms it as a PG 18 regression, and since 14e87ffa5 the cost reaches every partitioned schema with NOT NULL columns, whether or not it uses foreign keys.

### Precedents retrieved

- P1: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P2: [Prefetch the next tuple's memory during seqscans](https://commitfest.postgresql.org/patch/3978/) (cf, Withdrawn, 2024-01-22) - judged not relevant; reason summarised from its thread
- P3: [pg_get_indexdef modification to access catalog based on the TxnSnapshot](https://commitfest.postgresql.org/patch/4369/) (cf, Returned with feedback, 2024-01-05) - judged not relevant; reason summarised from its thread
- P4: [Shared system catalog cache](https://commitfest.postgresql.org/patch/2326/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P5: [auto_explain log_seqscan](https://commitfest.postgresql.org/patch/3086/) (cf, Returned with feedback, 2021-07-06) - judged not relevant; reason summarised from its thread
- P6: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P7: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P9: [Revert per-index collation version tracking feature.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec48314708262d8ea6cdcb83f803fc83dd89e721) (revert, Reverted, 2021-05-07) - judged not relevant
- P10: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P11: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/1252/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P12: [Foreign Key Arrays](https://commitfest.postgresql.org/patch/2966/) (cf, Returned with feedback, 2021-09-14) - judged not relevant
- P13: [Add ALWAYS DEFERRED option for CONSTRAINTs and CONSTRAINT TRIGGERs](https://commitfest.postgresql.org/patch/1319/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P14: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P15: [Add psql command to list constraints](https://commitfest.postgresql.org/patch/3468/) (cf, Rejected, 2022-03-28) - judged not relevant; reason summarised from its thread
- P16: [Reducing memory usage of creating foreign key between partitioned tables](https://commitfest.postgresql.org/patch/2330/) (cf, Rejected, 2020-09-12) - used in the review
- P17: [Revert temporal primary keys and foreign keys](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8aee330af55d8a759b2b73f5a771d9d34a7b887f) (revert, Reverted, 2024-05-16) - judged not relevant
- P18: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant
- P19: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P20: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [Revert "Catalog NOT NULL constraints" and fallout](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9ce04b50e120275afbc03ca0b80839dde3da8308) (revert, Reverted, 2023-04-12) - judged not relevant
- P22: [Temporal foreign key actions](https://commitfest.postgresql.org/patch/7238/) (cf, Withdrawn, 2026-09-17) - judged not relevant
- P23: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P24: [use SIMD in GetPrivateRefCountEntry()](https://commitfest.postgresql.org/patch/6101/) (cf, Withdrawn, 2025-12-15) - judged not relevant

### Warnings

- CommitFest entry has 3 threads; only the earliest is read (https://www.postgresql.org/message-id/flat/trinity-08d3329b-f6b2-4c11-91d7-2198e50bf4fc-1790681922354%40trinity-msg-rest-gmx-gmx-live-58cc8f554d-c56lm). Others: https://www.postgresql.org/message-id/flat/179089136170.117434.3768797469846977966%40gmail.com, https://www.postgresql.org/message-id/flat/CAApHDvpxGcb-_ddd%2BqL%2BF%3DVmv4d3WZNbUCi3WQyj0HcFnC9v1g%40mail.gmail.com

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 86 s; cost reported by the CLI: $1.65
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 49 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:45:46.927Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
