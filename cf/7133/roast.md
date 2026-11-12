# Design pre-review: Fix pg_basebackup long-path test on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7133](https://commitfest.postgresql.org/patch/7133/); [thread](https://www.postgresql.org/message-id/flat/CAAH-eSV4BJS7W%3DqRf7%3DHk09XD3NhFOVvvKRjH%2BLh0v9e_S%3DSeg%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible case in message #1: a 253-character path, an unchecked mkdir, and a misleading later error. The fix changes one TAP test file and has no user-visible surface. Confidence is medium because the code was not read and the v3 diff is not shown.

Not checked: The code and the v3 diff were not read. Two things could not be checked: whether the cleanup warnings come from this change, and whether the test still covers a long tablespace path.

## Prior attempts

None found before 2026-07-26.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the test fail on any buildfarm member or on CI today, or only when the build-tree path is long enough to pass 248 characters?
2. Do the directory-cleanup warnings in the v3 runs also appear without the patch, or does creating the directory through the short side of the junction cause them?
3. Should the fix be back-patched, and if so, to which stable branches does the same unchecked mkdir apply?
4. What are the "three patches" applied in the GitHub-hosted Windows x64 run, given that the thread attaches only one?
5. Does the test still exercise a long tablespace path after the directory is created through the short system temporary directory?

## Touch points

- **backpatchability:** The entry is tagged Bugfix for a test in 010_pg_basebackup.pl, and the thread does not say which stable branches need it.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson commented on v1 in message #2, and the demand review note describes him as a committer. Nobody has replied to v3 yet. The owner needs to know the TAP test framework and Windows path handling.
- Priority: proceed; audience niche; review cost low. The patch is about 1.5 KB in one test file and helps developers and buildfarm members who run the suite on Windows with long build paths. It adds no runtime code and makes a setup failure report at its source.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
