# Design pre-review: Backend crash (signal 11) in pg_trgm makesign() after ALTER TABLE ... SET STORAGE on a column with a gist_trgm_ops index

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7413](https://commitfest.postgresql.org/patch/7413/); [thread](https://www.postgresql.org/message-id/flat/CAA8TiqELrCMyAkNMev0NerxACZTaQL5a2Y2htzfDxFirJ%2BQGkw%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The problem is stated with a reproducer and a backtrace, and demand is shown: a backend crash on 16, 17 and 18. No objection survived at level 1a or 1b. The one surviving objection, standards-compat-1, is a fixable level 2 point: v1 stops new damage but the thread does not settle how users repair indexes that SET STORAGE already changed. A committer who knows index catalog code needs to decide the repair path and the back-branch plan.

Not checked: No patch was readable, so the exact type comparison, the SET COMPRESSION behaviour and the tests could not be checked. The REINDEX POC patch and message #4 were also not visible in full.

## Prior attempts

None found before 2026-10-02.

## Objections a reviewer is likely to raise

### 1. No defined repair path for indexes already changed by SET STORAGE

A user on 16 ran SET STORAGE EXTENDED last year, and their gist_trgm_ops index column still has attstorage 'x'. After the minor update with v1, the next insert still crashes, or an intarray index scan still loses rows, until someone repairs that index.

The v1 fix in message #2 only stops new damage, and the reporter found the bug on 16, 17, 18 and managed services, so damaged catalogs already exist in the field. Message #1 says REINDEX resets the column to 'p'. Message #4 says REINDEX fails the same way and adds a separate POC patch to reset the catalog values. The thread does not say whether the POC patch ships with v1, or what the release notes tell users to run.

**Question:** Does REINDEX on a damaged index work on released branches, and if not, does the reset patch go into every back branch together with v1?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAA8TiqELrCMyAkNMev0NerxACZTaQL5a2Y2htzfDxFirJ%2BQGkw%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAB8bMitfqZY32oVmB2ZgSRFA68HwpBYD5sKHtL0uKrwMyHNmMQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/CALdSSPjdKwQAX%2BQJd7%2BrcorAZd1GkqKZuTbjciYj8QpiKjf6%2Bg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Does REINDEX succeed on an index whose attstorage SET STORAGE already changed, given that message #1 says it does and message #4 says it fails?
2. Do the v1 fix and the REINDEX POC patch go in together, and into which back branches?
3. What exact step should the release notes tell users to run to repair an already affected index?
4. Do pg_upgrade and pg_dump carry the wrong attstorage, or the short-header keys, of an affected index to the new cluster?
5. Is comparing the index column type with the heap column type enough for every GiST, GIN and SP-GiST opclass with a STORAGE type, including third-party ones?

## Touch points

- **on-disk format / pg_upgrade:** Affected indexes already hold keys with short varlena headers, and v1 leaves those index pages as they are.
- **catalog changes:** The fix changes when SetIndexStorageProperties() updates pg_attribute.attstorage for index columns.
- **security / privileges:** Today a table owner can run SET STORAGE and make later inserts crash the backend; the fix removes that path.
- **backpatchability:** The crash is reported on 16, 17 and 18, and the proposal does not say which branches get the fix.
- **maintenance burden ("who maintains this in 10 years"):** The REINDEX POC patch adds a catalog reset step in REINDEX that must stay in step with the create-time storage rule.

## Who would commit it, and when

- Owner: silent. Someone who knows ALTER TABLE catalog updates in tablecmds.c and how index tuple descriptors take the opclass STORAGE type; no such committer has spoken in the visible thread.
- Priority: proceed; audience a sizeable minority; review cost medium. This is a reproducible backend crash and, per message #4, lost rows in index scans, for users of GiST, GIN or SP-GiST opclasses with a STORAGE type. The fix is one rule in one helper with no new user-visible surface; review cost is medium because no patch was read and the repair path is open.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
