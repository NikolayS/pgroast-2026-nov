# Design pre-review: EXPLAIN ANALYZE overcounts MERGE inserts suppressed by triggers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7606](https://commitfest.postgresql.org/patch/7606/); [thread](https://www.postgresql.org/message-id/flat/CAHewXNmZAG1i0PkRy%2Bb3koucghYn-ApBrYmtCgXDzYbRESA6Kg%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. Message #1 states the bug, gives a reproducer and explains why the caller cannot count correctly. The fix moves an existing counter and adds no user surface. Confidence is low because only one message is visible and the code was not read.

Not checked: Without the code, three points are unchecked: the ON CONFLICT and plain INSERT paths through ExecInsert(), the count for FDW batch rows the remote side rejects, and the cross-partition UPDATE check.

## Prior attempts

None found before 2026-11-06.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches have the wrong count, and do you propose to back-patch the fix?
2. How does ExecInsert() tell a MERGE insert action from a plain INSERT, INSERT ... SELECT or ON CONFLICT call, so that those paths leave mt_merge_inserted unchanged?
3. Is a row queued for FDW batch insertion counted as inserted if the remote side later rejects or skips it?
4. Do MATCHED UPDATE and DELETE actions report wrong counts when a BEFORE trigger returns NULL?

## Touch points

- **backpatchability:** The entry is tagged Bugfix, and a back-patch would change EXPLAIN ANALYZE counts for MERGE in released branches.
- **maintenance burden ("who maintains this in 10 years"):** ExecInsert(), shared by all insert paths, gains a check of the current MERGE action.
- **performance on the hot path:** The patch adds a check in ExecInsert(), which every inserted row passes through.

## Who would commit it, and when

- Owner: silent. A committer who knows the MERGE executor code in nodeModifyTable.c, including the cross-partition UPDATE and FDW batch insert paths, would have to own it. Nobody has replied yet.
- Priority: proceed; audience niche; review cost low. The bug affects users who read EXPLAIN ANALYZE counts for MERGE on tables with a BEFORE INSERT trigger that returns NULL. The patch is 4.3 KB, touches one executor file plus the merge regression test, and adds no user surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
