# Design pre-review: Fix out-of-bounds access in pg_bsd_indent's parser stack

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7424](https://commitfest.postgresql.org/patch/7424/); [thread](https://www.postgresql.org/message-id/flat/20260930105134.568062-1-vadimsakirov5%40gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated in message #1, and message #3 reproduces a segfault that filled a disk and confirms the patch turns it into a "Parser stack overflow" error. The fix changes two array sizes and one bound check in a developer tool, so it adds no cost for server users. Confidence is medium because no one has reviewed the code yet.

Not checked: Only the diff in message #1 was read, not the rest of parse.c. Whether every push in parse() and every use of il and cstk stays within the new bounds could not be checked.

## Prior attempts

None found before 2026-09-30.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches should get the fix, given that the CommitFest entry targets version 16?
2. Can any other path in parse() push more than two entries per call, so that the "- 2" bound is still too loose?
3. Should the reproducer from message #3 become a regression test for pg_bsd_indent?
4. Does upstream FreeBSD indent have the same defect, and if it is fixed there, does that fix match this one?

## Touch points

- **backpatchability:** The CommitFest entry targets version 16, but the thread does not say which branches should get the fix.
- **maintenance burden ("who maintains this in 10 years"):** The patch changes src/tools/pg_bsd_indent, code derived from FreeBSD indent, so it adds a small divergence from upstream.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson and Álvaro Herrera take part in the thread, and Álvaro tested the patch in message #3; neither has reviewed the code. The owner needs to know the pg_bsd_indent parser and pgindent.
- Priority: proceed; audience niche; review cost low. The patch changes three lines in two files of a developer tool and has a confirmed reproducer in message #3. It helps only developers who run pgindent, but it adds no user-visible surface and little to maintain.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
