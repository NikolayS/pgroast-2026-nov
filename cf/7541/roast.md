# Design pre-review: BUG #19703: information_schema.usage_privileges omits a sequence owner's implicit USAGE privilege

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7541](https://commitfest.postgresql.org/patch/7541/); [thread](https://www.postgresql.org/message-id/flat/19703-49d6f1796001a6d7%40postgresql.org), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at any level. The problem is stated with a reproducible case in message #1, the cause is identified in message #2, and the fix changes one acldefault type code in the sequences arm of the view. The thread agreed to fix master only, so no installed view changes on a minor upgrade. Confidence is medium because only 3 of 5 messages and no code were read.

Not checked: The code and messages #4 and #5 were not read. Whether the test covers the right cases and whether the fix touches only the sequences arm could not be checked.

## Prior attempts

None found before 2026-09-19.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do other arms of usage_privileges, or other information_schema views, pass an acldefault type code that does not match the object kind?
2. Should the commit bump catversion, as is usual for information_schema.sql changes?
3. Should the release notes tell users of existing installations how to replace the view definition?

## Touch points

- **catalog changes:** The patch edits a view definition in src/backend/catalog/information_schema.sql, and the catversion bump is undecided.
- **SQL-standard conflict or non-standard syntax:** The patch changes the rows returned by the SQL-standard view information_schema.usage_privileges.
- **security / privileges:** The view now reports the sequence owner's implicit USAGE; no access right changes.
- **backpatchability:** Message #3 raised a back-patch to 14; the thread chose master only because older branches need a manual view replacement.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane diagnosed the cause in message #2 and took part in the back-patch decision. The owner needs to know information_schema.sql and the ACL default codes.
- Priority: proceed; audience niche; review cost low. The fix is a small change to one file with a test, and a committer already confirmed the cause. Few users read sequence privileges through information_schema, but the view returns a wrong result today.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
