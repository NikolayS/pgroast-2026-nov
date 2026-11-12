# Design pre-review: Make ginInsertCleanup follow vacuum's BufferAccessStrategy

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7177](https://commitfest.postgresql.org/patch/7177/); [thread](https://www.postgresql.org/message-id/flat/76104756-9ca6-45ee-bd16-ead325f1c149%40app.fastmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. The patch extends the existing vacuum BufferAccessStrategy to the GIN pending-list reads, adds no user-visible surface, and leaves the insert path and gin_clean_pending_list() unchanged. Confidence is low because the thread is three messages long and shows no measurements.

Not checked: Without the code, it is not possible to check that every pending-list read in ginInsertCleanup() and shiftList() receives the strategy, or that the NULL callers behave as before. The effect of the ring buffer on a long pending list is unmeasured.

## Prior attempts

None found before 2026-08-18.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. What does the attached test script show, in buffer hits, evictions and vacuum duration, with and without the strategy?
2. Does the vacuum ring buffer slow cleanup of a long pending list, given that shiftList() re-reads the processed pages before deleting them?
3. Should gin_clean_pending_list() keep passing NULL, and what is the reason for treating it differently from the vacuum callers?
4. Do any known extensions call ginInsertCleanup() and need a change when its signature changes?

## Touch points

- **ABI / extension API:** ginInsertCleanup() in gin_private.h gains a BufferAccessStrategy argument.
- **backpatchability:** Message #3 says a backpatch to 14 is technically possible; a backpatch would change a function signature on stable branches.
- **performance on the hot path:** VACUUM of a GIN index now reads pending-list pages through the vacuum ring buffer instead of plain shared buffers.

## Who would commit it, and when

- Owner: silent. Someone who knows GIN fast-update cleanup and the vacuum buffer access strategy code would have to own it. The one reviewer in the thread agreed with the rationale only.
- Priority: proceed; audience niche; review cost low. The patch is 6.4 KB across three GIN files and adds no user-visible surface, so review is cheap. It benefits installations with large GIN pending lists; numbers from the test script would help a committer decide quickly.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
