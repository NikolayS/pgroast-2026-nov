# Design pre-review: Report unsupported COPY FROM targets before reading input

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6931](https://commitfest.postgresql.org/patch/6931/); [thread](https://www.postgresql.org/message-id/flat/84873FFF-4C50-4C85-9A36-62A01B860FF1%40gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because the moved relkind check accepts foreign tables and file_fdw passes with v3. The change moves an existing check to the place COPY TO already uses, and its cost falls only on commands that fail anyway. Confidence is medium because the code was not read and no committer has replied.

Not checked: The code was not read, so the exact position of the moved check among the other BeginCopyFrom() steps is unverified. Its effect on out-of-tree BeginCopyFrom() callers rests on the critic's reading and the reported file_fdw test result.

## Prior attempts

None found before 2026-06-24.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Will the patch add a regression test for the early failure, such as COPY of a sequence FROM a nonexistent file, as a reviewer suggested?
2. Should the commit message say that invalid COPY options such as (format bogus) are now reported after the relkind error?
3. Can an out-of-tree caller of BeginCopyFrom() pass a relation kind that the moved check rejects but that worked before because the caller never called CopyFrom()?
4. Which release does the patch target, given that the CommitFest entry says 19 and the thread tests on 20devel?

## Touch points

- **ABI / extension API:** BeginCopyFrom() is exported and used by file_fdw; it now rejects unsupported relation kinds itself.
- **psql UX conventions:** psql no longer enters COPY input mode for COPY FROM STDIN on a sequence, view or matview.
- **security / privileges:** COPY FROM PROGRAM on an unsupported target no longer runs the program; permission checks still come first.

## Who would commit it, and when

- Owner: silent. A committer who knows the COPY code in src/backend/commands and the BeginCopyFrom() callers (file_fdw, tablesync) would have to own it. None has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. The v3 patch is 4.8 KB in copyfrom.c, adds no user-visible surface and helps only users who run COPY FROM on an unsupported target. Three reviewers gave +1 over three CommitFests, but no committer has replied.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
