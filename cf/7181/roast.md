# Design pre-review: SQL/JSON boolean DEFAULT expression silently replaced with 'false'

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7181](https://commitfest.postgresql.org/patch/7181/); [thread](https://www.postgresql.org/message-id/flat/19625-683b498c92087bc8%40postgresql.org), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). This is a wrong-result fix in one parser function, and two bug reports show users hit it. The only surviving objection, replication-ondisk-1, is a fixable follow-up about views and rules that keep the stored 'false'::jsonb. It asks for release-note guidance and does not question the fix. A committer who knows the area plans to commit and back-patch it.

Not checked: Without the code, I could not check how ruleutils deparses the new to_jsonb() wrapper in a dumped view. Messages #4 and #5 were not shown, so the v2 details rest on the brief.

## Prior attempts

None found before 2026-08-17.

## Objections a reviewer is likely to raise

### 1. Stored views keep the wrong 'false'::jsonb after the fix, with no detection path

A user has a view with JSON_VALUE(... DEFAULT true ON ERROR) created on 17.x. They install the minor release with the fix, and later use pg_upgrade or dump and restore to move to v20. The view still returns false on error, and the user has no query that finds such views.

Message #5 says views and rules created before the fix keep the wrong 'false'::jsonb in their stored parse tree. The deparsed definition then shows DEFAULT 'false'::jsonb, so pg_dump, pg_upgrade and a physical standby all carry the wrong value forward as if the user wrote it. The fix is back-patched to v17, so many clusters may already hold affected rules. The thread does not say whether the release notes will tell users to recreate these views or how to find them.

**Question:** Will the release notes give a catalog query that finds affected views and rules, and tell users to recreate them before or after upgrading?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#5](https://www.postgresql.org/message-id/CA%2BHiwqGO9i5MUhOnnnXXrvKpou0A99wCfOuXrJcXRvTecmzTTg%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Will the release notes tell users to recreate views and rules that were created before the fix and hold the wrong 'false'::jsonb default?
2. Can the release notes give a catalog query that finds views and rules with a boolean DEFAULT in JSON_VALUE, JSON_QUERY or JSON_TABLE?
3. Does v2 add regression tests for JSON_TABLE column DEFAULT clauses with a boolean expression?
4. How does a new view with a boolean DEFAULT deparse after the fix, and does that text restore to the same behavior?

## Touch points

- **on-disk format / pg_upgrade:** Stored rules created before the fix keep 'false'::jsonb, and pg_upgrade carries them forward unchanged.
- **catalog changes:** No catalog change, but stored parse trees in pg_rewrite differ before and after the fix.
- **SQL-standard conflict or non-standard syntax:** The fix changes the result of SQL/JSON DEFAULT ... ON ERROR and ON EMPTY to the evaluated value.
- **pg_dump / restore round-trip:** Views created before the fix dump with the stored 'false'::jsonb default, so restore keeps the wrong value.
- **backpatchability:** The committer plans to back-patch to v17; v2 keeps the Const for canned TRUE and FALSE behaviors.

## Who would commit it, and when

- Owner: qualified committer engaged. Amit Langote posted v2 in the thread and says he intends to commit it and back-patch to v17.
- Priority: proceed; audience a sizeable minority; review cost low. The fix removes a silent wrong result for users of SQL/JSON functions with a boolean DEFAULT. It touches one function in parse_expr.c, and a committer who knows the area plans to commit it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
