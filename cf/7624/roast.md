# Design pre-review: Match jsonpath variable names exactly when checking mutability

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7624](https://commitfest.postgresql.org/patch/7624/); [thread](https://www.postgresql.org/message-id/flat/CAEEBa3VS8xnenaeLB_JUu2Nku6w6CLpaRvjf3BbXayfOt%2BeSbg%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection at levels 1a or 1b survived. Message #1 states a wrong-results bug with a reproduction, and the fix makes jspIsMutableWalker() match names as GetJsonPathVar() does. The one surviving objection, standards-compat-1, is fixable: a back-patch to 17 makes some existing index definitions fail at restore or pg_upgrade, and the proposal gives users no way to find them first. A committer who knows the SQL/JSON code should judge the back-patch.

Not checked: The code was not read, so the review cannot confirm that the new comparison equals the one in GetJsonPathVar(). It also cannot confirm that no other prefix comparison remains or that the back-patch applies to 17.

## Prior attempts

None found before 2026-11-10.

## Objections a reviewer is likely to raise

### 1. Back-patch makes existing dumps and pg_upgrade fail without a way to find them

A user on 17.x has a JSON_EXISTS index with PASSING names like aa and a. After a minor update, pg_restore of an older dump fails at create index. A later pg_upgrade to a new major version fails at the same index. The user finds out only when the restore or upgrade stops.

Message #1 and the commit message say restoring such definitions "will now fail at CREATE INDEX" and that REINDEX does not help. pg_upgrade restores index definitions through pg_dump, so it fails in the same way. Neither text offers a catalog query or release-note wording that lets a user find affected indexes before the update. A committer would likely accept the wrong-results fix but ask for this guidance before a minor-release back-patch.

**Question:** What query or release-note text will let 17.x users find affected index expressions before a minor update or a pg_upgrade?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAEEBa3VS8xnenaeLB_JUu2Nku6w6CLpaRvjf3BbXayfOt%2BeSbg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. What query or release-note text will let 17.x users find affected index expressions before a minor update or a pg_upgrade?
2. Does any code besides jspIsMutableWalker() still compare PASSING names by prefix?
3. Does the patch apply to the 17 and 18 branches as it is, or do those branches need a different version?

## Touch points

- **on-disk format / pg_upgrade:** pg_upgrade restores index definitions with create index, which now rejects indexes the old check accepted.
- **pg_dump / restore round-trip:** Message #1 says restoring a wrongly accepted index definition from a dump now fails at create index.
- **backpatchability:** The author asks for a back-patch to 17, which changes what create index accepts in a minor release.

## Who would commit it, and when

- Owner: silent. A committer who knows the SQL/JSON query functions and jsonpath code would have to own it; no one has replied to message #1.
- Priority: proceed; audience niche; review cost low. The bug gives wrong results from an index, but only for SQL/JSON index expressions where one PASSING name is a prefix of another. The patch is 5.0 KB, touches one source file and one regression test, and adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
