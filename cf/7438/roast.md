# Design pre-review: Fix grouping-set nulling of variable-free grouping expressions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7438](https://commitfest.postgresql.org/patch/7438/); [thread](https://www.postgresql.org/message-id/flat/CABXr29FSih8YtLCEe6Swquu_c%3DYvFiLVoH-ejvk2jVge9EMEMQ%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The bug is real and a fix is wanted, but the proposal now holds two definitions of the correct result (standards-compat-2). The commit message says `(1 + 1) * 10 ... rollup (1 + 1)` and now() cases are wrong results. The parser rules planned in message #6 stop matching such expressions, and message #8 accepts the unmatched result for now(). No posted patch contains those rules, so the list has to agree on the definition before design review.

Not checked: The parser rules exist only as a description in the thread, so their effect on currently correct queries and on stored views could not be judged. The planner code in var.c was not read.

## Prior attempts

None found before 2026-10-10.

## Objections a reviewer is likely to raise

### 1. Planned matching rules redefine the bug the patch reports

The v1/v3 commit message says `select (1 + 1) * 10 ... group by rollup (1 + 1)` should return NULL in the grand-total row. The second planned rule excludes grouping items without a level-zero Var from structural matching, so `1 + 1` inside `(1 + 1) * 10` is no longer a grouping reference and still returns 20. Users then cannot tell which result the project considers correct.

Message #8 accepts f,f for `now() is null ... rollup(now())` because the independently written expression is not a reference, but the commit message lists now() and constant expressions as cases that return wrong results. These two positions disagree on a user-visible contract, namely which expressions in the target list and HAVING count as grouping references. Rules that change query results in minor releases need one stated definition, checked against the standard and against 17 and 18 behaviour.

**Question:** Under the two planned parser rules, what does `select (1 + 1) * 10 from generate_series(1, 10) group by rollup (1 + 1)` return, and which documented rule says that is correct?

<sub>fixable, level 1a, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CABXr29FSih8YtLCEe6Swquu_c%3DYvFiLVoH-ejvk2jVge9EMEMQ%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CABXr29F4UDTaQVz7QZmVvLzz4R84ECP1%2Be6sVaSEWfbM6xKt%3Dw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAGRkXqTbwvQKNaGuWwOvH0-DMTvdrC_wBRpu9GzQC5dOzgDMcA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CABXr29EVOzjAseaNKO5xK2JrhQVP2UA89QfP3xw2A_gDOiNATA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Under the two planned parser rules, what does `select (1 + 1) * 10 from generate_series(1, 10) group by rollup (1 + 1)` return, and which stated rule makes that correct?
2. How does the back-patched planner avoid nulling group Vars that the old parser stored in existing views and rules on 18 and 19?
3. Which queries that return correct results on 18 today change results under the two parser rules, and is that acceptable in a minor release?
4. Can the two parser rules be back-patched to 18 and 19 in the same form as for master?
5. Will this fix be split from the patch for bug #19752, or wait for it?

## Touch points

- **catalog changes:** Views in pg_rewrite keep group Vars made by the old parser matching, and the new planner code acts on them.
- **ABI / extension API:** The patch adds a PlannerInfo field, to be moved to the end of the struct in back branches.
- **SQL-standard conflict or non-standard syntax:** The planned parser rules change which target-list and HAVING expressions count as references to a grouping item.
- **planner behaviour change:** mark_nullable_by_grouping() shares one PlaceHolderVar ID per RTE_GROUP column, which changes grouping-set query results.
- **pg_dump / restore round-trip:** A restore re-parses views under the new matching rules, so a view can return different rows than before the dump.
- **backpatchability:** The commit message asks for a back-patch through 18, and the fix now needs parser rules as well.
- **maintenance burden ("who maintains this in 10 years"):** The planner change in var.c is correct only while parse_agg.c matches genuine grouping references.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows grouping-set planning (RTE_GROUP, PlaceHolderVars, setrefs) and the grouping matching in parse_agg.c. The author waits for such an opinion, and none is in the visible thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. The 19 regression gives wrong rows for grouping sets over NOT NULL columns, so the fix deserves attention now. The posted patch is small, but v3 is known to be unsafe and the parser rules that complete it are not posted.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
