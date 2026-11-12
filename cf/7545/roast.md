# Design pre-review: BUG #19752: GROUP BY on a constant and a cast to a length-limited array type crashes the server while planning

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7545](https://commitfest.postgresql.org/patch/7545/); [thread](https://www.postgresql.org/message-id/flat/19752-928339cd23b58a33%40postgresql.org), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal or the rule in v3. The fix addresses a reported planner crash on Postgres 18, and a committer agreed in message #2 that wrong answers are also possible. The one surviving objection, replication-ondisk-1, is a fixable back-patch point about views already stored on 18.x. The remaining risk is whether the new rule is correct in the grouping code, which needs a committer who knows that code.

Not checked: Only 3 of 11 messages and the prose of the first patch were visible. Whether v3 handles outer references, subqueries and volatile items correctly cannot be judged without the code.

## Prior attempts

None found before 2026-10-08.

## Objections a reviewer is likely to raise

### 1. Views already stored on 18.x keep the bad substituted tree after the fix

A user on 18.4 creates a view whose query groups by a constant, such as `group by true` next to `ARRAY[]::varchar(500)[]`. pg_rewrite stores the parse tree with the grouped Var already substituted. After a minor update with this fix, selecting from the view still crashes or still gives the wrong result, because the stored rule is never parsed again.

The fix changes parse analysis in parse_agg.c, so it only affects trees built after the update. CREATE VIEW runs parse analysis but not planning, so the crashing tree from message #1 can already sit in pg_rewrite, as can the wrong-result trees Tom Lane mentions in message #2. pg_upgrade and dump/restore deparse and parse views again, so they repair them, but an in-place minor update does not. The thread does not say whether the fix targets 18, and it gives no note for users on what to recreate.

**Question:** For back-patched 18 releases, how will views and rules stored with substituted Var-free GROUP BY items be found and repaired, and will the release notes tell users to recreate them?

<sub>fixable, level 2, catalog changes; messages [#1](https://www.postgresql.org/message-id/19752-928339cd23b58a33%40postgresql.org), [#2](https://www.postgresql.org/message-id/2530849.1791487908%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/DMCBT3XQBZFK.2AR8PB94AGT40%40gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Is v3 meant for back-patching to 18, and if so, which query results on 18.x change after the minor update?
2. How will views and rules stored on 18.x with a substituted Var-free GROUP BY item be found, and will the release notes tell users to recreate them?
3. Does skipping Var-free items at every sublevel change results when the GROUP BY item is an outer reference or sits in a correlated subquery?
4. Do grouping sets queries with volatile Var-free items such as now() return the same results before and after v3?

## Touch points

- **catalog changes:** Views created on 18.x keep the substituted parse tree in pg_rewrite; the patch adds no catalog change itself.
- **planner behaviour change:** The patch changes which subexpressions parse_agg.c turns into grouped Vars, so the planner receives different trees.
- **security / privileges:** The fix removes a backend crash that any user can trigger without table access.
- **backpatchability:** The bug exists in released Postgres 18, and the thread does not say whether the fix goes back to 18.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a second matching rule for Var-free items to the grouping code in parse_agg.c.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane proposed the rule in the thread but has not commented on any patch version; the owner must know the RTE_GROUP substitution code in parse_agg.c.
- Priority: proceed; audience niche; review cost low. The trigger is narrow, but the result is a backend crash in a released branch, with possible wrong answers per message #2. The first patch is 4 KiB in parse_agg.c plus tests; v2 and v3 were not read.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
