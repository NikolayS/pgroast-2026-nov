# Design pre-review: Add support for INSERT ... SET syntax

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6635](https://commitfest.postgresql.org/patch/6635/) (Needs review); authors: Suraj Kharage (surajk)
- Thread: [\[PATCH\] Add support for INSERT ... SET syntax](https://www.postgresql.org/message-id/flat/CAF1DzPXhKc15Toa06MTT7NYOAEq8Ucr%3DRYe5V7X2FkwGMmjD6w%40mail.gmail.com), started 2026-03-30
- Messages read: 23 of 23
- Patch prose read: v7-0001-Add-support-for-INSERT-.-SET-syntax.patch, v1-0001-Add-support-for-INSERT-.-SET-syntax.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The single-row form has lasting demand and no surviving objection. The parenthesized multi-row form has no shown demand (demand-1) and no comparison with Oracle 23 or a draft standard text, while MariaDB spells it differently (prior-attempts-1). Its indirection rule turns an internal shared column list into a user-visible syntax rule (standards-compat-2). The list should settle the multi-row form before code review.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check grammar conflicts, the keyword class of SET, or how stored rules that use the new form deparse. Only 3 of 23 messages were read directly.

## The proposal, restated

**Problem.** In insert, the column list sits apart from the values. With many columns, or when only some columns get values, the reader must match names to values by position. The author wants a form that keeps each column name next to its value.

**Proposed design.** insert gains a SET clause as one more alternative to VALUES, DEFAULT VALUES and a query: `set col=expr, ...` for one row, or `set (col=expr, ...), (...)` for several rows. Rows may name different columns, columns are matched by name, and omitted columns get their default. In multi-row form, an array-element or subfield target must be identical in every row that touches that column, because all rows share one column list.

**User-visible surface.**
- New insert syntax: `insert into t set col=expr | DEFAULT [, ...]`, which cannot be combined with an explicit column list.
- New multi-row form: `insert into t set (col=expr, ...), (col=expr, ...)`, where rows may list different columns.
- Subfield and array-element targets (`c.x=1`, `arr[1]=1`). A repeated identical target in one row fails with "column ... specified more than once", which column-list insert accepts.
- New error when rows of a multi-row SET disagree on an indirection target for the same column.
- psql tab completion of SET and of column names after SET.

**What the author claims.**
- SET syntax is more readable than a separate column list and VALUES, mainly for single-row inserts of some columns.
- Works with DEFAULT, expressions, subqueries, RETURNING, ON CONFLICT and OVERRIDING { SYSTEM | USER } VALUE.
- Rejecting repeated identical targets ensures no assigned expression is silently discarded.
- Andrew Dunstan says Oracle 23 has this syntax and a proposal may exist to add it to the SQL standard.

**Thread state.** David Johnston gave -1 to inventing a variant insert syntax, Tom Lane pointed to earlier unadopted proposals, and Álvaro Herrera linked threads from 2016 and 2019. Reviewers then found a crash and array-target bugs that v7 addresses; Peter Eisentraut is the listed committer but has not posted, and Mario Gonzalez asks whether any committer supports it.

## 2. Prior attempts

### Implement INSERT SET syntax (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/2218/](https://commitfest.postgresql.org/patch/2218/)
- **Thread:** [https://www.postgresql.org/message-id/flat/385970D7-37BF-4D23-A5FD-525BBE18849E@internetnz.net.nz](https://www.postgresql.org/message-id/flat/385970D7-37BF-4D23-A5FD-525BBE18849E@internetnz.net.nz)
- **Date:** decided 2022-08-01; started 2019-07-17
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The patch got too little reviewer and committer interest. Tom Lane noted it had been ignored for two years, and the commitfest manager returned it because no one took up the review. Earlier concerns from Tom Lane and Peter are mentioned, but their messages are not shown.
- **Reason (quote):** "This patch has been basically ignored for a full two years now." ([message](https://www.postgresql.org/message-id/2720923.1648049571%40sss.pgh.pa.us))
- **What is different this time:** P2 also offered an optional FROM clause that turned the insert into INSERT ... SELECT. This patch drops FROM and adds a parenthesized multi-row form in which rows may name different columns.

### SET syntax for INSERT (same idea)

- **Link:** [https://commitfest.postgresql.org/patch/770/](https://commitfest.postgresql.org/patch/770/)
- **Date:** decided 2016-09-05; started 2016-08-31
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** The row records no design detail and no reason, so the material does not show how this proposal differs.

### MAP syntax for arrays (related)

- **Link:** [https://commitfest.postgresql.org/patch/1632/](https://commitfest.postgresql.org/patch/1632/)
- **Thread:** [https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)
- **Date:** decided 2018-07-19; started 2018-05-04
- **Outcome:** Rejected (cf)
- **Who objected:** Tom Lane, Andreas Karlsson, Ashutosh Bapat (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers objected to adding proprietary syntax that could conflict with future SQL standard use of the MAP keyword, which is already a non-reserved keyword in SQL:2016. They also judged the syntax not much better than the existing sub-select with unnest and array_agg, which is more flexible (for example, it allows a WHERE filter), and suggested optimizing the unnest/array_agg combination in the planner instead of adding new syntax. Heikki Linnakangas summarized the consensus as not wanting the feature and marked the entry Rejected.
- **Reason (quote):** "Yeah, it's a handy syntax, but it's not that much better than using a subselect or a writing a user-defined function. And there's risk of future conflicts with the SQL standard." ([message](https://www.postgresql.org/message-id/14ac5bbe-e9fb-2ce4-2d6d-698f8da29449%40iki.fi))
- **What is different this time:** P11 proposed different syntax (MAP over arrays) with no implementation elsewhere. Its concern about non-standard syntax that may conflict with a later standard applies here mainly to the multi-row form.


## 3. Objections a reviewer is likely to raise

### 1. Multi-row SET form may diverge from a future SQL standard

fixable · level 2, rubric item 6 · SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

Users write `insert ... set (a=1), (b=2)` in application code after release. If the standard later adopts a different multi-row form, such as MariaDB's repeated SET, Postgres must either carry two spellings forever or break those statements.

Message #4 says only that a standard proposal may exist, and nothing in the thread compares the patch's grammar with that text or with the Oracle form. Message #19 shows MariaDB already uses repeated SET for several rows, while this patch uses parenthesized groups, so the multi-row shape is not settled elsewhere. [P11] was rejected partly for the "risk of future conflicts with the SQL standard". Message #2 objects to Postgres inventing its own insert variant, and no reply answers that.

**Precedents.**
- [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (Rejected, 2018-07-19); thread: [link](https://www.postgresql.org/message-id/flat/f86e5d8b-20a7-36ce-acd6-ac58b290c8f0@postgrespro.ru)

Thread: [#2 ("David G(dot) Johnston", 2026-03-30)](https://www.postgresql.org/message-id/CAKFQuwbGSUXL5wfgPEnsGF3QTBHvOovNH9s1zUTUjv6Wocmk6w%40mail.gmail.com), [#4 (Andrew Dunstan, 2026-03-30)](https://www.postgresql.org/message-id/0238206d-81c9-4bce-acb4-127fbc4a96ac%40dunslane.net), [#19 (Vaibhav Dalvi, 2026-09-02)](https://www.postgresql.org/message-id/CA%2BvB%3DAEbDGcAiJ54iu0arRje3Wnm%2B%2BHUi4EmDQhXsgA6u-gaxQ%40mail.gmail.com)

**Question:** Does the parenthesized multi-row form match Oracle 23 or the draft standard text, and if not, why not limit the patch to the single-row form?

Counter-check: held. No message compares `set (...), (...)` with the Oracle form or a draft standard text, and message #19 reports a different multi-row spelling in MariaDB. [P11] concerned a new keyword with no prior implementation elsewhere, so it gives only weak support.

### 2. Same SET spelling follows different rules in insert and update

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user copies `set arr[1]=1, arr[1]=2` from an update into an insert ... set and gets "specified more than once". A multi-row insert where one row omits a column that another row assigns as `arr[1]` fails, while a single-row insert with the same omission succeeds, so the user cannot predict which statements work.

The v7 commit message says the duplicate rule is deliberately stricter than column-list insert and update. Message #20 adds a multi-row rule that exists only because rows share one column list internally. The user sees an internal parse-analysis shape as a syntax rule, a departure from existing practice that needs justification. A standard text may later define these cases differently.

Thread: [#15 (Vaibhav Dalvi, 2026-08-26)](https://www.postgresql.org/message-id/CA%2BvB%3DAH6FiOPpDiLd1%3DoC5TWmgKxEfT9Ha6u4yV1SJbUN7q_Lg%40mail.gmail.com), [#20 (Suraj Kharage, 2026-09-10)](https://www.postgresql.org/message-id/CAF1DzPWzx_uibbqEo2Bnj-aZYsv0J5ezE4E84NLam59G-vxgPQ%40mail.gmail.com), [#22 (Mario González, 2026-10-19)](https://www.postgresql.org/message-id/CAFsReFXn0-YLajW1Zg_kiAXL8UUSQ8owxXXPqQuavyNwY7Uhng%40mail.gmail.com), [#23 (Suraj Kharage, 2026-10-22)](https://www.postgresql.org/message-id/CAF1DzPXtdduFXWSJeShW0Em2y7fFNb09aopmk9bzttnpC97WOA%40mail.gmail.com)

**Question:** Why should insert ... set reject repeats that update ... set accepts, and what do Oracle or the standard proposal do in both cases?

Counter-check: held. The commit message justifies the duplicate-target rule, so that half is answered. The multi-row rule stands: message #20 says it exists because rows share one column list, so users see an internal limit as a syntax rule.

### 3. Multi-row set form has no shown demand and matches no cited system

fixable · level 1b, rubric item 3 · demand, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `demand-1`

A user ports a MariaDB multi-row insert that repeats SET per row and gets a syntax error. A later SQL standard or Oracle text may define multi-row SET differently. Postgres would then carry a parenthesized form that nobody asked for and that conflicts with the standard one.

The demand shown in the thread covers the single-row form: Oracle 23 (message #4), MySQL and MariaDB (message #19), and the 2016 and 2019 proposals (message #5). Message #19 reports that MySQL allows only one row and that MariaDB repeats SET for each row, which differs from `set (...), (...)`. No message shows that Oracle or the possible standard text uses the parenthesized form. Its restrictions on indirection targets, and the new error, exist only because of this form (messages #19, #20).

Thread: [#4 (Andrew Dunstan, 2026-03-30)](https://www.postgresql.org/message-id/0238206d-81c9-4bce-acb4-127fbc4a96ac%40dunslane.net), [#5 (Álvaro Herrera, 2026-03-30)](https://www.postgresql.org/message-id/202603301544.utwvuj534m36%40alvherre.pgsql), [#19 (Vaibhav Dalvi, 2026-09-02)](https://www.postgresql.org/message-id/CA%2BvB%3DAEbDGcAiJ54iu0arRje3Wnm%2B%2BHUi4EmDQhXsgA6u-gaxQ%40mail.gmail.com), [#20 (Suraj Kharage, 2026-09-10)](https://www.postgresql.org/message-id/CAF1DzPWzx_uibbqEo2Bnj-aZYsv0J5ezE4E84NLam59G-vxgPQ%40mail.gmail.com)

**Question:** Which existing system or draft standard text uses `set (col=expr, ...), (...)` for multi-row insert, and which users have asked for that form?

Counter-check: held. The demand in messages #4, #5 and #19 covers the single-row form; no message names a system or user request for `set (...), (...)`. Messages #19 and #20 confirm that the indirection restrictions come from this form alone.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes the raw InsertStmt node in parsenodes.h and adds handling in nodeFuncs.c. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds a SET alternative to insert, with a parenthesized multi-row form not compared with Oracle or the standard. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | The proposal does not say how ruleutils and pg_dump print rules that use insert ... set. |
| psql UX conventions | **yes** | The patch adds psql tab completion of SET and of column names after SET. |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A permanent second spelling of insert in gram.y and analyze.c, with duplicate and indirection rules that differ from column-list insert. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Do the single-row form and the parenthesized multi-row form match the Oracle 23 syntax and any draft SQL standard text, and where do they differ?
2. Which existing system or user request uses `set (col=expr, ...), (...)`, and if none, why not limit the patch to the single-row form?
3. Why must every row of a multi-row SET use the same subfield or subscript target, and can the patch remove that rule instead of documenting it?
4. Does the patch change the keyword class of SET or add grammar conflicts in gram.y?
5. How do ruleutils and pg_dump print a rule whose action uses insert ... set?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. A committer who knows gram.y and insert parse analysis must own it. Andrew Dunstan posted support and helped with v2; the committer listed on the CommitFest entry has not posted in the thread.
- **Liability:** The owner signs for the insert ... set grammar permanently, including the multi-row form and its indirection rules, even if a later SQL standard text spells them differently.
- **Author follow-up:** The author posted seven versions through message #23 and fixed the crash and the array-target bugs that reviewers reported.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The single-row form has demand from three Postgres proposals since 2016 and from Oracle 23, MySQL and MariaDB. The multi-row form needs a comparison with Oracle and the standard, or removal, before a review of the 49 KiB grammar and analysis patch is worth the time.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Multi-row SET spelling may not match Oracle or a future standard. Refuted (not specific): Duplicate of prior-attempts-1, which states the same multi-row divergence risk and adds a concrete way out (limit the patch to the single-row form).
- `standards-compat-3` Two non-positional insert forms proposed separately, with no shared design. Refuted (no failure scenario): The scenario depends on an INSERT BY NAME patch that is not yet submitted, and no concrete rule conflict is named. Message #6 shows its author knows this patch and chose to submit BY NAME separately.
- `maintenance-fit-1` SET form designed without the planned BY NAME variant or standard text. Refuted (not specific): Duplicate of prior-attempts-1 on the multi-row form. The BY NAME part repeats standards-compat-3 and has the same speculative scenario.
- `maintenance-fit-2` A second insert path with its own target rules to maintain. Refuted (already handled): Messages #18 and #20 say SET is merged into one shared column list and then goes through the existing transformAssignedExpr() and rewriter code. Defaults and OVERRIDING are handled in that shared code; the rule differences are covered by standards-compat-2.
- `maintenance-fit-3` No committer has engaged after three CommitFests. Refuted (wrong facts): Peter Eisentraut added himself as committer on the CommitFest entry, and Andrew Dunstan posted support and helped with v2 (messages #4, #8). The objection itself says it concerns committability, which the rubric keeps apart from the verdict.
- `already-exists-1` Column-list insert already does this; the gain is readability only. Refuted (already handled): Message #4 answers the -1 by pointing to Oracle 23, and message #5 lists independent proposals from 2016 and 2019 plus a user request. That is lasting demand from several sources; [P11] was syntax with no implementation elsewhere.
- `already-exists-2` Second by-name insert form planned in parallel with INSERT BY NAME. Refuted (not specific): Duplicate of prior-attempts-1 on the multi-row form. The BY NAME part names no concrete rule conflict, as in standards-compat-3.

### Review angles

- prior-attempts: 1 objection(s). [P2] ended for lack of reviewer and committer interest, and [P1] has no recorded reason. In message #3 Tom Lane asked why the earlier attempts were not adopted, and message #5 links them, but no message reports the answer. That gap affects committability, not the merits.
- standards-compat: 3 objection(s). The pg_dump and ruleutils round-trip was not raised. The new form likely becomes a normal insert during parse analysis, so a stored rule would print back as column-list syntax; checking this needs the code.
- replication-ondisk: 0 objection(s). The patch adds no persistent or cross-node state. The touched files are gram.y, analyze.c, parsenodes.h and nodeFuncs.c, so INSERT ... SET is turned at parse analysis into an ordinary insert Query with a column list and VALUES. That leaves no change to WAL, the on-disk format, the catalogs or the wire protocol. Stored rules should deparse as plain insert ... values, but this could not be checked without the code. The new raw-node field in InsertStmt is a master-only change.
- security: 0 objection(s). The proposal adds parser syntax that turns into an ordinary insert with a column list, so the existing table and column insert privileges, RLS policies and triggers apply unchanged; message #10 reports tests of column-level privilege violations. Nothing here creates a new privilege, default, exposure path or unbounded resource, so the security angle has no objection.
- maintenance-fit: 3 objection(s). Objections found; see list.
- already-exists: 2 objection(s). Two objections raised; no extension or external tool provides this, because it is grammar.
- demand: 1 objection(s). The single-row form has real demand: three Postgres proposals since 2016, an archive question from 2019, Oracle 23, MySQL and MariaDB. The earlier attempts [P1] [P2] stalled from lack of review, which is a committability issue rather than a demand issue.

### Precedents retrieved

- P1: [SET syntax for INSERT](https://commitfest.postgresql.org/patch/770/) (cf, Returned with feedback, 2016-09-05) - used in the review
- P2: [Implement INSERT SET syntax](https://commitfest.postgresql.org/patch/2218/) (cf, Returned with feedback, 2022-08-01) - used in the review; reason summarised from its thread
- P3: [Parallel INSERT (INTO ... SELECT ...)](https://commitfest.postgresql.org/patch/2844/) (cf, Returned with feedback, 2021-04-01) - judged not relevant; reason summarised from its thread
- P4: [Revert "Enable parallel SELECT for "INSERT INTO ... SELECT ..."."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=26acb54a1368bf3706294400abca85b15c9233a6) (revert, Reverted, 2021-03-24) - judged not relevant
- P5: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P6: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant; reason summarised from its thread
- P7: [Revert: Let table AM insertion methods control index insertion](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da841aa4dc279bb0053de56121c927ec943edff3) (revert, Reverted, 2024-04-11) - judged not relevant
- P8: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant; reason summarised from its thread
- P9: [Making autovacuum logs indicate if insert-based threshold was the triggering condition](https://commitfest.postgresql.org/patch/3802/) (cf, Withdrawn, 2022-11-27) - judged not relevant
- P10: [Ltree syntax improvement](https://commitfest.postgresql.org/patch/1977/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P11: [MAP syntax for arrays](https://commitfest.postgresql.org/patch/1632/) (cf, Rejected, 2018-07-19) - used in the review; reason summarised from its thread
- P12: [Revert Non text modes for pg_dumpall, and pg_restore support](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ce9a6244b5b4ce1df71611512a757353803404a5) (revert, Reverted, 2025-07-30) - judged not relevant
- P13: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P14: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P15: [Revert MERGE patch](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08ea7a2291db21a618d19d612c8060cda68f1892) (revert, Reverted, 2018-04-12) - judged not relevant
- P16: [table inheritance versus column compression and storage settings](https://commitfest.postgresql.org/patch/4813/) (cf, Returned with feedback, 2024-03-21) - judged not relevant
- P17: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P18: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P19: [Enable jitlink as an alternative jit linker of legacy Rtdyld and add riscv jitting support](https://commitfest.postgresql.org/patch/3857/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P20: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant
- P21: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant
- P22: [Inconsistency in owner assignment between INDEX and STATISTICS](https://commitfest.postgresql.org/patch/6571/) (cf, Withdrawn, 2026-03-11) - judged not relevant
- P23: [Revert "Fix bogus %name-prefix option syntax in all our Bison files."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71ed8b3ca76cd720f4013c3c20f0d2706583ab9a) (revert, Reverted, 2014-05-28) - judged not relevant
- P24: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 164 s; cost reported by the CLI: $3.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 102 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:57:12.853Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
