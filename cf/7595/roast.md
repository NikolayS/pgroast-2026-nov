# Design pre-review: MERGE fails to reject source references in NOT MATCHED BY SOURCE actions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7595](https://commitfest.postgresql.org/patch/7595/); [thread](https://www.postgresql.org/message-id/flat/CAHewXNmFA5dWp7AUivBjqLdjA3GyG4uVS2XDYdf%3DrChU-KMXGQ%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). No surviving objection questions the goal or the design: the patch makes an unaliased join source follow the rule that a single-table source already follows. The one surviving objection, standards-compat-1, is a fixable back-patch question that a committer who knows the MERGE parser must decide. Confidence is low because only message #1 exists and the code was not read.

Not checked: Without the code, it cannot be checked that the per-action namespace is restored on every path, or that aliased joins, subqueries, nested joins and NOT MATCHED BY TARGET actions keep their current behaviour.

## Prior attempts

None found before 2026-11-05.

## Objections a reviewer is likely to raise

### 1. Back-patch policy for newly rejected MERGE queries is not stated

An application runs a MERGE with an unaliased join source that references a source column in a NOT MATCHED BY SOURCE action. After a minor-release update, that MERGE fails with "invalid reference to FROM-clause entry". The user finds out when the batch job errors in production.

The patch turns queries that parse and run today into parse errors, and it is tagged as a bugfix. Message #1 does not say which branches have the bug or whether the fix goes back. Rejecting these queries on master is consistent with the single-table case. On a stable branch, a committer would likely weigh the error against the current silent behaviour, where the source column has no matching source row and so cannot hold a source value.

**Question:** Which branches should get this fix, and how does it weigh breaking running queries on a minor update against the silent result those queries give today?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAHewXNmFA5dWp7AUivBjqLdjA3GyG4uVS2XDYdf%3DrChU-KMXGQ%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which released branches accept these source references today, and do you propose to back-patch the fix or apply it to master only?
2. If back-patched, how do you weigh a new parse error on a minor update against the result these MERGE statements give today?
3. How do an aliased join, a subquery and nested joins in USING behave before and after the patch?
4. Do NOT MATCHED BY TARGET and MATCHED actions see the same namespace after the patch as before, including target references?

## Touch points

- **SQL-standard conflict or non-standard syntax:** The patch changes which MERGE statements with a join source pass parse analysis; it adds no syntax.
- **backpatchability:** The entry is tagged Bugfix, and a back-patch would make MERGE statements that run today fail at parse time.

## Who would commit it, and when

- Owner: silent. Someone who knows MERGE parse analysis and the parser namespace rules in transformFromClause() would have to own it; no one has replied to message #1.
- Priority: proceed; audience niche; review cost low. The patch is small, touches only parse_merge.c and the merge regression test, and adds no user-visible surface beyond an error. It affects only MERGE with an unaliased join source and a NOT MATCHED BY SOURCE action.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
