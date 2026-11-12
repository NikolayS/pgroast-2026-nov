# Design pre-review: pgstat: Add helper to drop entries from a stats kind's hash

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7546](https://commitfest.postgresql.org/patch/7546/); [thread](https://www.postgresql.org/message-id/flat/CAN12%2BYL7SrAwzMpzfNLrVcy%3Dw5L03p7czyO73RfS06kmdqs_fw%40mail.gmail.com), 18 of 18 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The change adds one argument to an internal function, on master only, and old callers fail at compile time. A committer who knows the area chose the single-function design and reports committing v6 in message #18. Confidence is medium because only 3 of 18 messages and no code were read.

Not checked: The code was not read, so the claim that the kind filter runs before the match callback and before the NULL-callback drop-all rests on the commit message and message #3. Messages #4 to #18 were not shown in full.

## Prior attempts

None found before 2026-10-27.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does any known extension call pgstat_drop_matching_entries() today, and should the release notes mention the new argument?
2. What match callback will the pg_stat_statements follow-up pass, and does it need anything beyond the kind filter?
3. Is there a measurement of the time saved when a kind with its own dshash skips the shared variable stats hash?

## Touch points

- **ABI / extension API:** pgstat_drop_matching_entries() in pgstat_internal.h gains a PgStat_Kind argument, so extension callers must change their call.
- **backpatchability:** The commit message says the signature change breaks ABI, so it fits master only.

## Who would commit it, and when

- Owner: qualified committer engaged. Michael Paquier reviewed the patch, chose the single-function design, and writes in message #18 that he committed v6.
- Priority: proceed; audience niche; review cost low. The patch is about 10 KiB, touches one backend file plus a test module, and serves authors of custom stats kinds. Message #18 reports it committed, so the CommitFest entry, still marked Needs review, can be updated.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
