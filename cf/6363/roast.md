# Design pre-review: Add a berief general comment on BTScanInsertData's nextkey and backward

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6363](https://commitfest.postgresql.org/patch/6363/); [thread](https://www.postgresql.org/message-id/flat/20251118162801.27cb13408cb3c9dd3a72cf7d%40sraoss.co.jp), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patches change comments only in nbtree.h and nbtsearch.c, and Peter Geoghegan called the first comment reasonable in message #2. The remaining risk is whether the v2-0001 wording is accurate, and the material does not show that text.

Not checked: The new comment text in v2-0001 and v2-0002 was not read, so its accuracy against _bt_binsrch and _bt_first could not be checked.

## Prior attempts

None found before 2025-11-18.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the new _bt_binsrch comment in v2-0001 use the pivot tuple and downlink explanation from message #2, or a different one?
2. Is the suggested edit to the comment at the end of _bt_first, from message #3, part of v2?
3. Does v2-0002 change the nbtree.h wording from the first version?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** New comments in nbtree.h and nbtsearch.c must stay accurate when _bt_first or _bt_binsrch change.

## Who would commit it, and when

- Owner: qualified committer engaged. Peter Geoghegan replied in message #2 and asked for the _bt_binsrch change; a committer who knows nbtree search must confirm the v2-0001 wording.
- Priority: needs champion; audience niche; review cost low. Two small comment-only patches help readers of the nbtree code and add no runtime cost. The entry has waited since March 2026 for a reply to v2 from someone who knows nbtree.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
