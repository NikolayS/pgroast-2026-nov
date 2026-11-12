# Design pre-review: Clarify wait versus poll behavior in ExecAppendAsyncEventWait

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7612](https://commitfest.postgresql.org/patch/7612/); [thread](https://www.postgresql.org/message-id/flat/C05FA422-F3E6-4198-AB7F-70570C157001%40gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at any level. The patch changes comments only in nodeAppend.c, and Etsuro Fujita, who reviewed v1 in message #2, rewrote it as v2 and, per the brief, pushed it on 2026-11-10. Confidence is medium because the final v2 comment text and the commit are not in the material.

Not checked: The v2 comment text and the pushed commit are not in the material, so the accuracy of the final wording could not be checked.

## Prior attempts

None found before 2026-11-05.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which commit and which branches received the v2 comment change?
2. Can the CommitFest entry be marked Committed, since it still shows Needs review and Patch needs rebase?

## Touch points

None of the 15 is hit.

## Who would commit it, and when

- Owner: qualified committer engaged. Etsuro Fujita reviewed v1, wrote v2 in message #2, and pushed it per the brief. The owner needs to know the async Append executor code.
- Priority: proceed; audience niche; review cost low. The change helps developers who read the async Append code and costs nothing at run time. The brief says it was pushed on 2026-11-10, so the only remaining step is to update the CommitFest entry.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
