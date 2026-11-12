# Design pre-review: doc: Mention that * does not include system columns

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7599](https://commitfest.postgresql.org/patch/7599/); [thread](https://www.postgresql.org/message-id/flat/178782026936.2606573.10270614257871556098%40wrigleys.postgresql.org), 10 of 10 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The change rewrites one sentence on the SELECT reference page to state existing behaviour, and Peter Eisentraut's placement concern in message #3 was addressed by moving the text there. Confidence is medium because only the v1 hunk is shown, so the v4 wording cannot be checked.

Not checked: Only the v1 hunk for syntax.sgml is shown. The v4 text, its exact place on the SELECT reference page, and whether it covers table_name.* could not be checked.

## Prior attempts

None found before 2026-08-27.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the v4 sentence on the SELECT reference page also cover table_name.*, which v2 named and v3 rewrote?
2. Do other pages that describe * expansion, such as composite types or row constructors, need the same note or a link to this one?

## Touch points

None of the 15 is hit.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Eisentraut reviewed v1 in message #3 and chose the target page, but has not commented on v4; a committer who handles documentation would sign it.
- Priority: proceed; audience a sizeable minority; review cost low. A reader asked for this through the docs comment form (message #1), and the change is one sentence plus a link on one page. A reviewer gave +1 on v4; it needs a committer to read the final wording.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
