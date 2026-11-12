# Design pre-review: pg_combinebackup: Don't point only to pg_checksums for mixed checksum states

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7587](https://commitfest.postgresql.org/patch/7587/); [thread](https://www.postgresql.org/message-id/flat/anwm6UPxoVS41QA2%40bdtpg), 44 of 44 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. The patch rewords one pg_combinebackup hint and one documentation paragraph and leaves the refusal as it is. Confidence is low because the material shows neither the new hint wording nor the new docs text, and nobody has replied to message #44.

Not checked: The new hint wording and the new docs text were not in the material, so their accuracy and clarity could not be checked. Whether the hint still gives the user a next step depends on that wording.

## Prior attempts

None found before 2026-08-12.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. What is the exact new hint text, and does it still tell the user what to do next, for example take a new full backup?
2. Should this change go into 19 together with the refusal, given that the CommitFest entry targets version 20?
3. Should the Limitations paragraph say that the refusal is stricter than needed after an online enable?
4. Does the new docs text say what the user should do when the full backup was taken while checksums were still being enabled?

## Touch points

- **backpatchability:** The refusal comes from the v19 open-item work, but the CommitFest entry targets version 20.

## Who would commit it, and when

- Owner: silent. Someone who knows pg_combinebackup and the v19 data checksums work would own it. Committers took part earlier in the thread, but none has replied to message #44.
- Priority: proceed; audience niche; review cost low. Only users who change the checksum state inside an incremental backup chain see this hint, but for them the current text names a tool they may never have run. The patch changes one string and one docs paragraph, so review is short.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
