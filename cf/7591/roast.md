# Design pre-review: BUG? Unpublished generated columns are reported "missing" by the subscription

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7591](https://commitfest.postgresql.org/patch/7591/); [thread](https://www.postgresql.org/message-id/flat/CAHut%2BPu7CuuqSJNzXUXiBfhByd16j4JtLXeNEr%3DQtPMjjH02Rg%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. Both raised objections (replication-ondisk-1, maintenance-fit-1) concerned committing 0002 without 0001, and the critic refuted them because 0002's own tests fail without 0001. The proposal fixes a confirmed v18 regression in tablesync, and two reviewers approved v2-0002. Open points are the back-patch plan and the review of 0001, which happens in another thread.

Not checked: The code was not read, and only the first 3 of 7 messages were shown. Whether the NULL-attrs check handles explicit column lists and mixed publisher versions correctly, and whether 0001 resolves the empty list safely, could not be checked.

## Prior attempts

None found before 2026-11-04.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Will the fix be back-patched to v18, the release that introduced the bug, and will 0001 and 0002 go to the same branches together?
2. Is adding the exported BeginCopyFromExtended() acceptable in back branches, or does the back-branch fix need a different form?
3. When a v18+ publisher has an explicit column list, how does the subscriber tell NULL attrs meaning "no columns published" from NULL meaning "no column list"?
4. Must 0002 wait until 0001 is committed from its own thread, and who reviews 0001?

## Touch points

- **WAL / replication / standby:** 0002 changes how the tablesync worker reads NULL attrs from pg_get_publication_tables() for v18+ publishers.
- **ABI / extension API:** 0001 adds the exported function BeginCopyFromExtended() to commands/copy.h; BeginCopyFrom() stays unchanged.
- **backpatchability:** The bug started in v18, but the CommitFest entry targets version 20 and the thread states no back-patch plan.
- **maintenance burden ("who maintains this in 10 years"):** COPY FROM gets a second entry point, BeginCopyFromExtended(), with tablesync as its only caller today.

## Who would commit it, and when

- Owner: silent. A committer who knows logical replication tablesync and the COPY FROM setup code would have to own it; none has spoken in the visible thread.
- Priority: proceed; audience niche; review cost low. The bug is a confirmed v18 regression that affects only tables whose columns are all unpublished generated columns. Both patches are small, touch tablesync and COPY FROM setup, and 0002 already has two approvals; 0001 still needs review in its own thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
