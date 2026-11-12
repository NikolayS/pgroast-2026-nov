# Design pre-review: Docs: Create table description for constraints markup fix and label tweaks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6648](https://commitfest.postgresql.org/patch/6648/); [thread](https://www.postgresql.org/message-id/flat/CAKFQuwbsGWx5msYbRsWrRs%2Bss1k0utDHEhVq5unAKODQ9YOf9w%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. Both patches change only doc/src/sgml/ref/create_table.sgml: 0001 corrects element nesting for UNIQUE and PRIMARY KEY, and 0002 answers a reader report on -general cited in message #1. A reviewer confirmed in message #3 that make html builds. The remaining points are scope and wording choices for a committer.

Not checked: The documentation hunks were read in full, but the rendered HTML was not seen. Whether "variant" is the best wording, and whether the same labels exist on other pages, could not be checked from the material.

## Prior attempts

None found before 2026-04-03.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Should the 0001 markup fix be back-patched, and if so to which branches?
2. Do other reference pages, such as alter table, use the "(column constraint)" and "(table constraint)" labels and need the same change?
3. Is the inconsistent use of optional markup versus literal brackets, noted in message #2, in scope for this patch set or left for later?
4. Does the original reporter on -general find that "(column constraint variant)" removes the confusion with the column_constraint syntax name?

## Touch points

- **backpatchability:** The CommitFest entry is tagged Bugfix, and the thread does not say whether the 0001 markup fix goes to back branches.

## Who would commit it, and when

- Owner: silent. Any committer who handles documentation patches and knows the reference page markup conventions could take this; none has replied in the thread.
- Priority: needs champion; audience most users; review cost low. The two patches are a few lines in one widely read reference page and already have a positive review in message #3. The entry has moved through two CommitFests without a committer reply, so it needs a committer to pick it up.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
