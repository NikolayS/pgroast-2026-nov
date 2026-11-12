# Design pre-review: BUG? Unpublished generated columns are reported "missing" by the subscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7591](https://commitfest.postgresql.org/patch/7591/) (Needs review); authors: Ajin Cherian (ajinc)
- Thread: [BUG? Unpublished generated columns are reported "missing" by the subscription](https://www.postgresql.org/message-id/flat/CAHut%2BPu7CuuqSJNzXUXiBfhByd16j4JtLXeNEr%3DQtPMjjH02Rg%40mail.gmail.com), started 2026-11-04
- Messages read: 7 of 7
- Patch prose read: v1-0001-Fix-initial-sync-of-zero-column-published-tables.patch, v1-0002-Fix-tablesync-of-tables-with-no-published-columns.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. Both raised objections (replication-ondisk-1, maintenance-fit-1) concerned committing 0002 without 0001, and the critic refuted them because 0002's own tests fail without 0001. The proposal fixes a confirmed v18 regression in tablesync, and two reviewers approved v2-0002. Open points are the back-patch plan and the review of 0001, which happens in another thread.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and only the first 3 of 7 messages were shown. Whether the NULL-attrs check handles explicit column lists and mixed publisher versions correctly, and whether 0001 resolves the empty list safely, could not be checked.

## The proposal, restated

**Problem.** Since v18, initial table synchronization fails with "logical replication target relation ... is missing replicated column" when every column of a published table is an unpublished generated column. If the subscriber has a regular column of the same name, tablesync copies generated values that incremental replication never sends. It affects logical replication users with such tables, and only when copy_data is true.

**Proposed design.** Tablesync also fetches whether pg_get_publication_tables() returned NULL attrs, and for v18+ publishers treats NULL as "no columns published" instead of "all columns". This leaves zero replicated columns, so the fix depends on a separate zero-column fix (0001) that adds BeginCopyFromExtended() so an explicitly empty COPY column list differs from an unspecified one.

**User-visible surface.**
- C API: new BeginCopyFromExtended() in commands/copy.h, from the bundled 0001 patch; BeginCopyFrom() is unchanged.
- Tablesync of tables with no published columns now succeeds and fills subscriber-only columns with their defaults.

**What the author claims.**
- Older publishers are not affected because they exclude generated columns in the pg_attribute query.
- Without 0001, text COPY loads '' instead of the local default and binary COPY fails with "row field count is 0".
- The existing BeginCopyFrom() interface and behavior stay unchanged.
- The patch is AI assisted.

**Thread state.** Shveta Malik confirmed the bug, Peter Smith reviewed v1 and approved v2-0002, and Shlok Kyal tested v2-0002 and approved it. No one reviewed 0001 in this thread; it belongs to a separate thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0002 changes how the tablesync worker reads NULL attrs from pg_get_publication_tables() for v18+ publishers. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 adds the exported function BeginCopyFromExtended() to commands/copy.h; BeginCopyFrom() stays unchanged. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The bug started in v18, but the CommitFest entry targets version 20 and the thread states no back-patch plan. |
| maintenance burden ("who maintains this in 10 years") | **yes** | COPY FROM gets a second entry point, BeginCopyFromExtended(), with tablesync as its only caller today. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Will the fix be back-patched to v18, the release that introduced the bug, and will 0001 and 0002 go to the same branches together?
2. Is adding the exported BeginCopyFromExtended() acceptable in back branches, or does the back-branch fix need a different form?
3. When a v18+ publisher has an explicit column list, how does the subscriber tell NULL attrs meaning "no columns published" from NULL meaning "no column list"?
4. Must 0002 wait until 0001 is committed from its own thread, and who reviews 0001?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows logical replication tablesync and the COPY FROM setup code would have to own it; none has spoken in the visible thread.
- **Liability:** The owner signs for the changed reading of NULL attrs in tablesync across publisher versions, and for a second COPY FROM entry point, in every branch that receives the fix.
- **Author follow-up:** The author posted a v2 after Peter Smith's review of v1, which shows follow-up during review; no sign either way for after commit.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug is a confirmed v18 regression that affects only tables whose columns are all unpublished generated columns. Both patches are small, touch tablesync and COPY FROM setup, and 0002 already has two approvals; 0001 still needs review in its own thread.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Committing 0002 without 0001 turns a loud error into wrong subscriber data. Refuted (already handled): Message #3 says the fix depends on the zero-column fix and bundles it as 0001 so that the tests pass. The 0002 tests cover text and binary mode, so 0002 pushed alone to any branch fails its own tests.
- `maintenance-fit-1` Fix needs a second COPY FROM entry point reviewed in another thread. Refuted (no failure scenario): The failure scenario duplicates replication-ondisk-1, and the 0002 tests guard against it. The entry-point concern names no concrete failure: the 0001 commit message says BeginCopyFrom() stays unchanged, and a new exported function does not break ABI in a back branch.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P7] added publish_generated_columns and closed idle over how virtual generated columns should replicate, which does not apply to this tablesync bugfix. The thread links only to the zero-column COPY discussion, as the source of 0001 (messages #1, #3, #5), and reports no earlier rejection of this fix.
- standards-compat: 0 objection(s). No standards or compatibility objection holds. The fix adds no syntax, GUC, catalog object or psql change, and makes tablesync match incremental replication. The C API is additive, since BeginCopyFrom() keeps its behaviour. Back-patch status and the dependency on 0001 are open, but no failure scenario follows from them for existing users.
- replication-ondisk: 1 objection(s). The proposal does not change the on-disk format, WAL, the catalogs or the wire protocol. The new BeginCopyFromExtended() only adds a function. The v18+ publisher version check covers subscribers that connect to older publishers.
- security: 0 objection(s). The proposal has no new privilege or exposure aspect. It adds no new role, GUC, SQL entry point or default. Subscriber-only columns get their defaults through the usual COPY path, which already runs the subscriber's default expressions and triggers. The fix also stops tablesync from copying generated values that the publication does not publish, which narrows exposure. BeginCopyFromExtended() is an internal C API and not reachable from SQL.
- maintenance-fit: 1 objection(s). This is a narrow, confirmed bug fix in tablesync. It has no hot-path cost and adds nothing users must configure, and two logical replication reviewers approved 0002. The only maintenance concern is its dependency on 0001.
- already-exists: 0 objection(s). This is a bug fix in tablesync for a v18 regression. No existing feature, option or extension gives correct initial sync for tables whose only columns are unpublished generated columns. copy_data = false avoids the error but skips the initial copy, so it does not do the same job. BeginCopyFromExtended() follows the usual practice of adding a variant entry point and keeping the old one stable, and it does not duplicate a separate mechanism.
- demand: 0 objection(s). This is a confirmed v18 regression in tablesync that can copy generated values incremental replication never sends, so demand evidence is not the test. The affected tables are rare, but a correctness fix carries no user-facing cost, and [P7] concerns the feature itself, not this bug.

### Precedents retrieved

- P1: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P2: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant; reason summarised from its thread
- P3: [foreign key on virtual generated column](https://commitfest.postgresql.org/patch/5782/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P4: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P5: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant; reason summarised from its thread
- P6: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant; reason summarised from its thread
- P7: [Allow logical decoding to output generated columns](https://commitfest.postgresql.org/patch/5068/) (cf, Returned with feedback, 2024-12-10) - used in the review; reason summarised from its thread
- P8: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P9: [Logical Replication - revisit `is_table_publication` function implementation](https://commitfest.postgresql.org/patch/7042/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P10: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant
- P11: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant
- P12: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P13: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P14: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant
- P15: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P16: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P17: [Warn if initdb's --sync-only option is mixed with other options](https://commitfest.postgresql.org/patch/3239/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [empty](https://commitfest.postgresql.org/patch/5037/) (cf, Rejected, 2024-06-21) - judged not relevant
- P19: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P20: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P21: [Revert "Allow logical replication snapshots to be database-specific"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=01a80f062146af1b17b411c35cb8d992c487fa7c) (revert, Reverted, 2026-05-24) - judged not relevant
- P22: [Fix NULL dereference in pg_get_database_ddl()](https://commitfest.postgresql.org/patch/6669/) (cf, Withdrawn, 2026-04-13) - judged not relevant
- P23: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant
- P24: [Add missing function abs (interval)](https://commitfest.postgresql.org/patch/3054/) (cf, Returned with feedback, 2021-12-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 107 s; cost reported by the CLI: $1.23
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 53 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:20.948Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
