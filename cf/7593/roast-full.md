# Design pre-review: Fix initial sync of zero-column published tables

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7593](https://commitfest.postgresql.org/patch/7593/) (Needs review); authors: Mingjie Gao (mingjie.gmj)
- Thread: [\[PATCH v1\] Fix initial sync of zero-column published tables](https://www.postgresql.org/message-id/flat/CAL5_ZS1bnYkK_GrA4N4SVKYgYsMMJxL9416m%3DvECAjrcwmQkkg%40mail.gmail.com), started 2026-11-01
- Messages read: 7 of 7
- Patch prose read: v1-0001-Fix-initial-sync-of-zero-column-published-tables.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The thread states the bug with a reproducer, and Peter Smith reproduced it and confirmed the fix in message #3. The fix adds one C entry point used only by tablesync and leaves BeginCopyFrom() unchanged. Confidence is medium because the code was not read and the backpatch range is still open.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the review could not check that the explicit-list flag reaches every place where COPY FROM resolves the column list and prepares defaults. Behavior on back branches is untested.

## The proposal, restated

**Problem.** When a published table has zero columns and the subscriber table has extra columns, initial table synchronization mishandles those columns. In text mode COPY stores an empty string instead of the subscriber default, while later incremental inserts use the default. In binary mode the initial copy fails with "row field count is 0, expected 1".

**Proposed design.** Tablesync passes an empty column list, which BeginCopyFrom() reads as "all columns". The patch adds BeginCopyFromExtended() with a flag that marks the column list as explicit, and tablesync uses it so an empty list stays empty. BeginCopyFrom() keeps its behavior for SQL COPY, file_fdw and other callers.

**User-visible surface.**
- C API: new function BeginCopyFromExtended() in commands/copy.h, with a flag for an explicit column list.
- Behavior: initial sync of a zero-column published table now fills subscriber-only columns with their defaults, in text and binary modes.

**What the author claims.**
- There are no SQL syntax or replication protocol changes, and the patch does not change which publisher columns are replicated.
- New tests in 008_diff_schema.pl fail without the fix and pass with it, in text and binary modes.
- Related subscription, COPY and file_fdw tests pass, and the patch was tested on master only.
- The text-mode bug reproduces on PG10, and the binary case likely affects PG16 and later.
- The patch should be backpatched to all affected supported branches.

**Thread state.** Peter Smith reproduced the bug, confirmed the fix, and gave comment and test-style feedback, and vignesh C asked for test changes and the backpatch range. The author posted v3 with all suggestions applied; the backpatch decision is still open and no committer has replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-01; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch changes tablesync.c, so initial sync of a logical replication subscriber behaves differently for zero-column published tables. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds the exported function BeginCopyFromExtended() to commands/copy.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The author asks for a backpatch to all affected branches, but tested master only and the range is not settled. |
| maintenance burden ("who maintains this in 10 years") | **yes** | COPY FROM gets a second entry point, BeginCopyFromExtended(), that future changes to BeginCopyFrom() must keep in step. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which supported branches show the text-mode bug and which show the binary-mode failure, and does the patch apply and pass its tests on each of them?
2. Should the back branches get the new exported function BeginCopyFromExtended(), or a fix that stays inside tablesync.c, with the API change on master only?
3. Can any other caller of BeginCopyFrom() pass an empty column list and mean "no columns" instead of "all columns"?
4. What should users do with rows that an earlier initial sync already stored with an empty string, and will the release note say so?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows logical replication tablesync and the COPY FROM code would have to own it; none has replied in the visible thread.
- **Liability:** The owner signs for a new exported COPY entry point on every branch that gets the fix, and for a behavior change in initial sync on stable branches.
- **Author follow-up:** The author posted a reproducer in message #2 and, per the thread state, a v3 that applies all review suggestions within about a week.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug needs a zero-column published table and extra subscriber columns, which is rare, but it stores wrong data in text mode and fails in binary mode. The patch is 7.5 KB in four files and reviewed by two people; it waits for a committer and a backpatch decision.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the corpus search returned no earlier proposal with the same or a similar idea, and no thread message points to an earlier discussion. From the prior-attempts angle there is no recorded rejection reason to apply to this bug fix.
- standards-compat: 0 objection(s). From the standards and compatibility angle there is no objection. The patch adds no SQL syntax, GUC, catalog state, dump output or protocol change. It fixes a subscriber-only divergence: initial sync stored '' where later inserts used the subscriber default. Adding BeginCopyFromExtended() beside an unchanged BeginCopyFrom() follows the existing *Extended naming pattern, and on back branches it adds a symbol without changing an existing signature. A committer would likely confirm two points. First, the fix runs on the subscriber, so the publisher version does not matter. Second, rows already synced with '' stay as they are after a minor upgrade, and the release note could say so.
- replication-ondisk: 0 objection(s). The fix changes only how the subscriber reads an empty column list, so no WAL, protocol, catalog or on-disk change follows, and mixed publisher versions behave the same. On back branches the patch adds BeginCopyFromExtended() and leaves BeginCopyFrom() unchanged, which keeps the ABI; rows already synced with '' stay as they are and need a release note.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. With the fix, tablesync evaluates subscriber default expressions for subscriber-only columns, which the apply worker already does for incremental inserts under the same role, so no new code runs and no new data or privilege becomes reachable. The new BeginCopyFromExtended() is a C entry point and adds nothing callable from SQL.
- maintenance-fit: 0 objection(s). This is a narrow bug fix with a reproducer, confirmed in message #3, and it adds nothing to the hot path. A committer may still ask whether master should take a different API shape than the back branches, but no concrete failure scenario follows from adding BeginCopyFromExtended(). Committability: no committer has replied, although the author keeps answering review.
- already-exists: 0 objection(s). This is a bug fix for tablesync. No existing feature, option or extension lets tablesync pass an explicitly empty column list to COPY, so the need is not met elsewhere. An *Extended() variant of an existing function follows common Postgres practice. The back-branch ABI and backpatch-range questions fall under other angles.
- demand: 0 objection(s). This is a bug fix, so demand is not the test: initial sync stores '' where incremental apply stores the default, and binary sync fails outright. Peter Smith reproduced it independently (message #3). The cost is one internal COPY entry point used only by tablesync.

### Precedents retrieved

- P1: [Warning for Datatype mismatch between subscriber table and publisher table for logical replication](https://commitfest.postgresql.org/patch/5648/) (cf, Returned with feedback, 2025-05-13) - judged not relevant; reason summarised from its thread
- P2: [DOCS - Generated Column Logical Replication Examples](https://commitfest.postgresql.org/patch/5590/) (cf, Withdrawn, 2025-02-21) - judged not relevant; reason summarised from its thread
- P3: [Multi-column list partitioning](https://commitfest.postgresql.org/patch/3383/) (cf, Returned with feedback, 2022-02-07) - judged not relevant; reason summarised from its thread
- P4: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P5: [empty](https://commitfest.postgresql.org/patch/5037/) (cf, Rejected, 2024-06-21) - judged not relevant
- P6: [Time-delayed logical replication subscriber](https://commitfest.postgresql.org/patch/3581/) (cf, Returned with feedback, 2023-03-10) - judged not relevant; reason summarised from its thread
- P7: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P8: [Table AM modifications to accept column projection lists](https://commitfest.postgresql.org/patch/2922/) (cf, Withdrawn, 2021-06-30) - judged not relevant; reason summarised from its thread
- P9: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/3556/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P10: [skip empty transactions for logical replication](https://commitfest.postgresql.org/patch/2384/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P11: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P12: [Logical Replication - revisit `is_table_publication` function implementation](https://commitfest.postgresql.org/patch/7042/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P13: [Useless field ispartitioned in CreateStmtContext](https://commitfest.postgresql.org/patch/5340/) (cf, Rejected, 2024-11-28) - judged not relevant
- P14: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P15: [speed up a logical replication setup (pg_subscriber)](https://commitfest.postgresql.org/patch/4636/) (cf, Withdrawn, 2023-11-01) - judged not relevant
- P16: [Optimize LISTEN/NOTIFY](https://commitfest.postgresql.org/patch/5912/) (cf, Withdrawn, 2025-08-07) - judged not relevant
- P17: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant
- P18: [Support empty ranges with bounds information](https://commitfest.postgresql.org/patch/3028/) (cf, Rejected, 2021-03-03) - judged not relevant
- P19: [Allow logical replication via inheritance root table](https://commitfest.postgresql.org/patch/4225/) (cf, Returned with feedback, 2023-08-31) - judged not relevant
- P20: [AT add columns with default in O(1)](https://commitfest.postgresql.org/patch/843/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P21: [Revert "Allow logical replication snapshots to be database-specific"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=01a80f062146af1b17b411c35cb8d992c487fa7c) (revert, Reverted, 2026-05-24) - judged not relevant
- P22: [Fix creation of empty .log files during log rotation](https://commitfest.postgresql.org/patch/5365/) (cf, Rejected, 2026-01-08) - judged not relevant
- P23: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - judged not relevant
- P24: [Allow LISTEN on patterns](https://commitfest.postgresql.org/patch/5630/) (cf, Withdrawn, 2025-03-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 109 s; cost reported by the CLI: $1.09
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 44 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:03:44.376Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
