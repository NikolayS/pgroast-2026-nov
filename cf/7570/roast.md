# Design pre-review: Reject wholerow references where ColumnRef is forbidden

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7570](https://commitfest.postgresql.org/patch/7570/); [thread](https://www.postgresql.org/message-id/flat/CA%2BrenyXoVubKLxGf%3Dopu4ty_OYbEC1TkpX5rGfwDQpUiAga-%3DA%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived. The only one raised, replication-ondisk-1, was refuted by the critic. The proposal fixes server crashes that message #1 reproduces, Tom Lane agreed it is a bug in message #2, and v2 moves the check into expandNSItemVars as he asked. The remaining points are code-review questions about the v14 and v15 versions.

Not checked: Without the code, this review cannot confirm that the v14/15 patches cover every caller of expandNSItemVars, or that the check on master rejects no legal expansion. It also cannot confirm the critic's claim that stored defaults dump as plain column references.

## Prior attempts

None found before 2026-10-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. On v14 and v15, which callers of expandNSItemVars do not call checkColumnRefAllowed, and why is each of them safe?
2. Which expression kinds besides column defaults and partition bounds forbid column references, and does a regression test reach each one through the new path?
3. With 0001 alone, what error message does CREATE TABLE p1 PARTITION OF p FOR VALUES IN ((ROW(p1.*)).f1) give?
4. Is patch 0002, which only changes an error message, meant to be back-patched to v14 as its commit message says?
5. Does the check inside expandNSItemVars on master reject any expansion of a table's columns that is legal today?

## Touch points

- **ABI / extension API:** The patch adds checkColumnRefAllowed to parser/parse_expr.h and, on master, changes how expandNSItemVars is called; v14 and v15 keep its exported signature.
- **SQL-standard conflict or non-standard syntax:** Defaults and partition bounds with a whole-row reference such as (ROW(t.*)).f1 now give an error instead of a crash.
- **security / privileges:** The fix removes a server crash that any user allowed to create a table can cause.
- **backpatchability:** The commit messages say Backpatch-through: 14, with separate v14/15 patches that put the check in each caller of expandNSItemVars.
- **maintenance burden ("who maintains this in 10 years"):** On v14 and v15 the check sits in each caller of expandNSItemVars, the pattern Tom Lane called omission-prone in message #2.
- **performance on the hot path:** On master, expandNSItemVars runs an expression-kind check on every expansion of a table's columns; the cost is one test per expansion.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane, who knows the parser, reviewed v1 in message #2 and asked for the change that v2 makes. He has not replied to v2.
- Priority: proceed; audience niche; review cost low. Few users write such defaults, but the statements in message #1 crash the server on every branch back to v14. The patches are small, touch only the parser and regression tests, and a committer who knows the area is engaged.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
