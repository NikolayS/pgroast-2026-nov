# Design pre-review: Improve cube GiST page splits

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7196](https://commitfest.postgresql.org/patch/7196/); [thread](https://www.postgresql.org/message-id/flat/3F6C9E73-9A6C-409D-9734-3D7C4E7FEA81%40yandex-team.ru), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because each GiST opclass already keeps its own picksplit. The patch fixes a measured defect in an existing contrib opclass, copies the box and point approach, and adds no user-visible surface. Confidence is medium because nobody has reviewed v2 and no numbers exist for v2.

Not checked: The code was not read, so the v2 handling of NaN, infinite and extreme coordinates could not be checked. Split quality of v2 and split CPU cost at many dimensions are also unverified.

## Prior attempts

None found before 2026-08-09.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does v2 keep the v1 page counts and buffers per query on the eight data shapes from message #2?
2. How does insertion time and CREATE INDEX time compare with master for random, unsorted input?
3. What is the CPU cost of one split at high dimensionality, up to the cube maximum of dimensions?
4. Is there any input that master indexes and v2 still rejects with an ERROR, beyond the cases covered by the new tests?
5. Can the commit message state that existing cube GiST indexes need no reindex and that old and new splits can coexist in one index?

## Touch points

- **on-disk format / pg_upgrade:** New splits change the shape and size of cube GiST indexes; the tuple and page format stay the same.
- **pg_dump / restore round-trip:** Message #2 showed v1 made CREATE INDEX fail on restore for NaN and large coordinates; v2 says it fixes this.
- **maintenance burden ("who maintains this in 10 years"):** v2-0002 adds an N-dimensional double-sorting split to contrib/cube/cube.c, separate from the box and point code.
- **performance on the hot path:** The patch replaces the picksplit that runs on every cube GiST page split, for cube users only.

## Who would commit it, and when

- Owner: silent. A committer who knows GiST picksplit and the float8 edge cases (NaN, infinity, overflow) in the geometric opclasses would have to own it. None has spoken in the visible thread.
- Priority: proceed; audience niche; review cost medium. The patch touches one file in contrib/cube plus tests and adds no SQL objects, GUCs or C API, and message #1 shows a very large gain for sorted input. Review cost is medium because the float8 edge cases need care and v2 is unreviewed.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
