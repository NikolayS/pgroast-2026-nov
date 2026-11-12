# Design pre-review: Add support for INSERT ... SET syntax

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6635](https://commitfest.postgresql.org/patch/6635/); [thread](https://www.postgresql.org/message-id/flat/CAF1DzPXhKc15Toa06MTT7NYOAEq8Ucr%3DRYe5V7X2FkwGMmjD6w%40mail.gmail.com), 23 of 23 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The single-row form has lasting demand and no surviving objection. The parenthesized multi-row form has no shown demand (demand-1) and no comparison with Oracle 23 or a draft standard text, while MariaDB spells it differently (prior-attempts-1). Its indirection rule turns an internal shared column list into a user-visible syntax rule (standards-compat-2). The list should settle the multi-row form before code review.

Not checked: Without the code, this review could not check grammar conflicts, the keyword class of SET, or how stored rules that use the new form deparse. Only 3 of 23 messages were read directly.

## Prior attempts

- [Implement INSERT SET syntax](https://www.postgresql.org/message-id/flat/385970D7-37BF-4D23-A5FD-525BBE18849E@internetnz.net.nz) (same idea; Returned with feedback, 2022-08-01). "This patch has been basically ignored for a full two years now." ([message](https://www.postgresql.org/message-id/2720923.1648049571%40sss.pgh.pa.us)) Different now: P2 also offered an optional FROM clause that turned the insert into INSERT ... SELECT. This patch drops FROM and adds a parenthesized multi-row form in which rows may name different columns.
- [SET syntax for INSERT](https://commitfest.postgresql.org/patch/770/) (same idea; Returned with feedback, 2016-09-05). reason not recorded Different now: The row records no design detail and no reason, so the material does not show how this proposal differs.
- [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (related; Rejected, 2018-07-19). "Yeah, it's a handy syntax, but it's not that much better than using a subselect or a writing a user-defined function. And there's risk of future conflicts with the SQL standard." ([message](https://www.postgresql.org/message-id/14ac5bbe-e9fb-2ce4-2d6d-698f8da29449%40iki.fi)) Different now: P11 proposed different syntax (MAP over arrays) with no implementation elsewhere. Its concern about non-standard syntax that may conflict with a later standard applies here mainly to the multi-row form.

## Objections a reviewer is likely to raise

### 1. Multi-row SET form may diverge from a future SQL standard

Users write `insert ... set (a=1), (b=2)` in application code after release. If the standard later adopts a different multi-row form, such as MariaDB's repeated SET, Postgres must either carry two spellings forever or break those statements.

Message #4 says only that a standard proposal may exist, and nothing in the thread compares the patch's grammar with that text or with the Oracle form. Message #19 shows MariaDB already uses repeated SET for several rows, while this patch uses parenthesized groups, so the multi-row shape is not settled elsewhere. [P11] was rejected partly for the "risk of future conflicts with the SQL standard". Message #2 objects to Postgres inventing its own insert variant, and no reply answers that.

Precedent: [MAP syntax for arrays](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru) (Rejected, 2018-07-19)

**Question:** Does the parenthesized multi-row form match Oracle 23 or the draft standard text, and if not, why not limit the patch to the single-row form?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#2](https://www.postgresql.org/message-id/CAKFQuwbGSUXL5wfgPEnsGF3QTBHvOovNH9s1zUTUjv6Wocmk6w%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/0238206d-81c9-4bce-acb4-127fbc4a96ac%40dunslane.net), [#19](https://www.postgresql.org/message-id/CA%2BvB%3DAEbDGcAiJ54iu0arRje3Wnm%2B%2BHUi4EmDQhXsgA6u-gaxQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Same SET spelling follows different rules in insert and update

A user copies `set arr[1]=1, arr[1]=2` from an update into an insert ... set and gets "specified more than once". A multi-row insert where one row omits a column that another row assigns as `arr[1]` fails, while a single-row insert with the same omission succeeds, so the user cannot predict which statements work.

The v7 commit message says the duplicate rule is deliberately stricter than column-list insert and update. Message #20 adds a multi-row rule that exists only because rows share one column list internally. The user sees an internal parse-analysis shape as a syntax rule, a departure from existing practice that needs justification. A standard text may later define these cases differently.

**Question:** Why should insert ... set reject repeats that update ... set accepts, and what do Oracle or the standard proposal do in both cases?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#15](https://www.postgresql.org/message-id/CA%2BvB%3DAH6FiOPpDiLd1%3DoC5TWmgKxEfT9Ha6u4yV1SJbUN7q_Lg%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAF1DzPWzx_uibbqEo2Bnj-aZYsv0J5ezE4E84NLam59G-vxgPQ%40mail.gmail.com), [#22](https://www.postgresql.org/message-id/CAFsReFXn0-YLajW1Zg_kiAXL8UUSQ8owxXXPqQuavyNwY7Uhng%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/CAF1DzPXtdduFXWSJeShW0Em2y7fFNb09aopmk9bzttnpC97WOA%40mail.gmail.com); `standards-compat-2`</sub>

### 3. Multi-row set form has no shown demand and matches no cited system

A user ports a MariaDB multi-row insert that repeats SET per row and gets a syntax error. A later SQL standard or Oracle text may define multi-row SET differently. Postgres would then carry a parenthesized form that nobody asked for and that conflicts with the standard one.

The demand shown in the thread covers the single-row form: Oracle 23 (message #4), MySQL and MariaDB (message #19), and the 2016 and 2019 proposals (message #5). Message #19 reports that MySQL allows only one row and that MariaDB repeats SET for each row, which differs from `set (...), (...)`. No message shows that Oracle or the possible standard text uses the parenthesized form. Its restrictions on indirection targets, and the new error, exist only because of this form (messages #19, #20).

**Question:** Which existing system or draft standard text uses `set (col=expr, ...), (...)` for multi-row insert, and which users have asked for that form?

<sub>fixable, level 1b, demand; messages [#4](https://www.postgresql.org/message-id/0238206d-81c9-4bce-acb4-127fbc4a96ac%40dunslane.net), [#5](https://www.postgresql.org/message-id/202603301544.utwvuj534m36%40alvherre.pgsql), [#19](https://www.postgresql.org/message-id/CA%2BvB%3DAEbDGcAiJ54iu0arRje3Wnm%2B%2BHUi4EmDQhXsgA6u-gaxQ%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/CAF1DzPWzx_uibbqEo2Bnj-aZYsv0J5ezE4E84NLam59G-vxgPQ%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Do the single-row form and the parenthesized multi-row form match the Oracle 23 syntax and any draft SQL standard text, and where do they differ?
2. Which existing system or user request uses `set (col=expr, ...), (...)`, and if none, why not limit the patch to the single-row form?
3. Why must every row of a multi-row SET use the same subfield or subscript target, and can the patch remove that rule instead of documenting it?
4. Does the patch change the keyword class of SET or add grammar conflicts in gram.y?
5. How do ruleutils and pg_dump print a rule whose action uses insert ... set?

## Touch points

- **ABI / extension API:** The patch changes the raw InsertStmt node in parsenodes.h and adds handling in nodeFuncs.c.
- **SQL-standard conflict or non-standard syntax:** The patch adds a SET alternative to insert, with a parenthesized multi-row form not compared with Oracle or the standard.
- **pg_dump / restore round-trip:** The proposal does not say how ruleutils and pg_dump print rules that use insert ... set.
- **psql UX conventions:** The patch adds psql tab completion of SET and of column names after SET.
- **maintenance burden ("who maintains this in 10 years"):** A permanent second spelling of insert in gram.y and analyze.c, with duplicate and indirection rules that differ from column-list insert.

## Who would commit it, and when

- Owner: qualified committer engaged. A committer who knows gram.y and insert parse analysis must own it. Andrew Dunstan posted support and helped with v2; the committer listed on the CommitFest entry has not posted in the thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. The single-row form has demand from three Postgres proposals since 2016 and from Oracle 23, MySQL and MariaDB. The multi-row form needs a comparison with Oracle and the standard, or removal, before a review of the 49 KiB grammar and analysis patch is worth the time.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
