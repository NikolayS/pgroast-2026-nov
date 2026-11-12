# Design pre-review: Fix trigger memory leak

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7569](https://commitfest.postgresql.org/patch/7569/); [thread](https://www.postgresql.org/message-id/flat/CA%2BrenyVvS8K0YypfwYfRNT8-8OHoDKPPNv%3DpcSC0RdZx%2BuV__w%40mail.gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated, and message #3 measures a statement-lifespan leak of about 104 MiB for 50k rows through views and partitioned tables. The fix is confined to trigger.c and adds no user-visible surface. Confidence is medium because the safety of the added free depends on code that was not read.

Not checked: Without the code, it could not be checked that the freed tuple is never referenced afterwards, or that the free happens only when the slot made a copy. The four functions named in the commit message were not compared with other trigger paths.

## Prior attempts

None found before 2026-10-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Can a C trigger function keep or return a pointer into the tuple that the patch now frees, and if so, what guards against use after free?
2. Does the patch apply to every branch back to 14, or do some branches need a different version?
3. Do other trigger paths, such as delete triggers or triggers on foreign tables, leak the same copied tuple?
4. Does the patch add a regression test, and if not, how would a later change that brings the leak back be noticed?

## Touch points

- **ABI / extension API:** The patch frees a tuple that a C trigger function in an extension may have received or returned.
- **backpatchability:** The commit message asks for a back-patch through 14.
- **performance on the hot path:** The patch adds a per-row free in the BEFORE and INSTEAD OF row trigger paths.

## Who would commit it, and when

- Owner: silent. Someone who knows executor tuple slots and trigger.c tuple ownership would have to own it. No committer has replied in the visible thread; the commit message credits Andres Freund only as reporter.
- Priority: proceed; audience a sizeable minority; review cost low. The leak reaches users of INSTEAD OF triggers on views and BEFORE INSERT triggers on partitioned tables, at about 104 MiB per 50k rows in message #3. The patch is about 3 KiB in one file, with two reviewers approving.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
