# Design pre-review: Add a warning on stats manipulation functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6155](https://commitfest.postgresql.org/patch/6155/); [thread](https://www.postgresql.org/message-id/flat/TY3PR01MB118914FCCBBDC126C34AA287382B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The patch adds one paragraph to an existing docs warning and changes no code. Message #1 shows demand from several PGEcons member companies, and message #2 supports each factor the paragraph names. The only surviving objection, standards-compat-1, is about where the note is placed and is fixable. Wording and the missing rebase are documentation review, outside levels 1-2.

Not checked: Only the first 3 of 13 messages and the v1 docs hunk were read. The exact wording, the SGML placement after a rebase, and any agreement in later messages could not be checked.

## Prior attempts

None found before 2025-04-08.

## Objections a reviewer is likely to raise

### 1. Warning sits on the functions page, not where pg_dump users look

A user follows the workflow from message #1 and runs pg_dump --statistics-only, then restores the output on staging. They read the pg_dump and pg_restore pages, which do not carry the new warning. They find out only when staging plans differ from production.

The question in message #1 is about pg_dump --statistics-only, and message #3 asked for the note on the pg_dump page. The patch in message #8 changes only func-admin.sgml, which pg_dump users do not need to read, because pg_dump calls pg_restore_relation_stats() for them. No later message settles where the note belongs, so the patch may not stop the repeat question it was written for.

**Question:** Why does the warning not also belong in, or link from, the pg_dump and pg_restore pages that describe --statistics-only?

<sub>fixable, level 1b, pg_dump / restore round-trip; messages [#1](https://www.postgresql.org/message-id/TY3PR01MB118914FCCBBDC126C34AA287382B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), [#3](https://www.postgresql.org/message-id/TY3PR01MB11891C649F6B4406994EC972982B52%40TY3PR01MB11891.jpnprd01.prod.outlook.com), [#8](https://www.postgresql.org/message-id/20251023192753.165621c1ae24b04b718cebf4%40sraoss.co.jp); `standards-compat-1`</sub>

## Questions for the author

1. Should the pg_dump and pg_restore pages that describe --statistics-only carry the same note, or a link to it, as message #3 asked?
2. Do you accept Tanaka's wording from message #13: "actual relation file sizes", "the creation order of indexes", and the actual data range read from indexes?
3. Should the warning also name work_mem and the other real-data effects that Corey Huinker raised, or say that the list of factors is not complete?
4. Can you post a rebased version, since CFbot reported in 2026-03 that the patch needs a rebase?

## Touch points

- **pg_dump / restore round-trip:** The warning is about statistics restored from pg_dump --statistics-only output, but the pg_dump and pg_restore pages get no note.

## Who would commit it, and when

- Owner: silent. A committer who knows the statistics import and export documentation would have to pick this up; the visible messages show no committer review of the patch.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is a 738-byte docs change with demand shown in message #1 and almost no cost after commit. It needs a rebase and a revision that settles the wording and the placement before a committer can take it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
