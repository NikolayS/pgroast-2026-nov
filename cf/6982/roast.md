# Design pre-review: Proposal: INSERT ... BY NAME

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6982](https://commitfest.postgresql.org/patch/6982/); [thread](https://www.postgresql.org/message-id/flat/CAJTYsWVDs7vkEN-eD1NAnZ7XqcfpXu3tP1wMGzJQ12G6eo1oRw%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the parse-analysis design. The one surviving objection, standards-compat-1, is fixable: the patch accepts BY on either side of OVERRIDING and BY POSITION with DEFAULT VALUES, and nobody has checked these forms against the draft standard text. A committer who knows the grammar and the standard would need to sign off on the released syntax.

Not checked: Without the code, this review cannot confirm that the Query node is unchanged, how names are compared, or which columns get the insert privilege check when no column list is given.

## Prior attempts

- [Corresponding by clause for set operation](https://www.postgresql.org/message-id/flat/CALAY4q8XqRqAaNsdC2Om+r8dNEgZbaBHTQYT+8DL0BCRdo7H0w@mail.gmail.com) (similar idea; Returned with feedback, 2017-04-04). "The main thing that's scaring me off is a realization that there are a *lot* of places that assume that the output columns of a set operation are one-for-one with the columns of the inputs." ([message](https://www.postgresql.org/message-id/30942.1491069285%40sss.pgh.pa.us)) Different now: P4 matched set-operation inputs by name and broke code that assumes set-operation outputs map one-for-one to inputs. This proposal resolves BY NAME in parse analysis into an ordinary positional insert with a reordered target column list, a form INSERT already supports.

## Objections a reviewer is likely to raise

### 1. Grammar commitments go beyond the unverified draft standard text

The published standard fixes BY NAME to one position relative to OVERRIDING, or does not allow BY POSITION with DEFAULT VALUES. Postgres has already released both orders and the no-op form. It must then keep the non-standard spellings for compatibility, or break user scripts that rely on them.

No one has checked the patch against the draft text. Jim Jones said he would try to get it (message #7), and Andreas Karlsson confirmed only the explicit column list case (messages #4 and #5). The patch accepts the clause on either side of OVERRIDING and accepts BY POSITION with DEFAULT VALUES. The thread does not say whether the draft allows either form. Syntax extensions are hard to remove once released, so a committer would likely want each accepted form tied to the draft grammar before commit.

**Question:** Which of the accepted forms, both OVERRIDING orders and BY POSITION with DEFAULT VALUES, does the draft standard grammar allow?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#4](https://www.postgresql.org/message-id/52c9a352-0e4a-4eaa-a9e1-06fd33cf562f%40proxel.se), [#5](https://www.postgresql.org/message-id/c5090d60-09b3-4ca3-97cc-ddb99d87a301%40proxel.se), [#7](https://www.postgresql.org/message-id/349ee7d0-ed22-41a3-8d1f-698cc0170e99%40uni-muenster.de), [#12](https://www.postgresql.org/message-id/CAJTYsWWrRNhAYQvt-p9puKU5CGj2bc5HU9Jajm4gKzcsKz4BBg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which of the accepted forms, both OVERRIDING orders and BY POSITION with DEFAULT VALUES, does the draft standard grammar allow?
2. Does name matching follow identifier case rules, for example a quoted source alias `"C1"` against an unquoted target column c1?
3. How does BY NAME behave with generated columns, identity columns, ON CONFLICT and RETURNING?
4. Without a target column list, does the patch check insert privilege only on the columns that the source query names?
5. Does the Query node gain a new field, or is the change limited to the raw InsertStmt?

## Touch points

- **ABI / extension API:** The patch changes the raw InsertStmt node in parsenodes.h, which extensions that inspect parse trees can see.
- **SQL-standard conflict or non-standard syntax:** The patch adds `BY { NAME | POSITION }` to INSERT from a draft standard that nobody in the thread has read.
- **pg_dump / restore round-trip:** Rules and SQL function bodies that use BY NAME are stored and dumped in resolved positional form.
- **psql UX conventions:** v2 adds tab completion for BY NAME and BY POSITION in tab-complete.in.c.
- **backpatchability:** see objection standards-compat-1: Grammar commitments go beyond the unverified draft standard text
- **maintenance burden ("who maintains this in 10 years"):** New INSERT grammar forms in gram.y become permanent syntax once released.

## Who would commit it, and when

- Owner: silent. Someone who knows gram.y and parse analysis of INSERT, and who can read the draft SQL standard text. The visible thread and the CommitFest entry show no such committer taking part.
- Priority: proceed; audience a sizeable minority; review cost low. The patch is about 29 KiB, touches only the parser, docs, psql completion and tests, and resolves to an ordinary positional insert. The open point is a check of the accepted forms against the draft standard, which does not need a redesign.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
