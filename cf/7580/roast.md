# Design pre-review: doc: Fix description of relative extension "directory" setting

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7580](https://commitfest.postgresql.org/patch/7580/); [thread](https://www.postgresql.org/message-id/flat/CABkiuWpGiLzDg4D8zRBJ6DHnwYgHqkFxz2gDstXgeOdzh6bdZQ%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The patches change documentation and one GUC description string so that they match the code and the TAP test already shipped in 18, and message #1 gives a reproducible case. The remaining open points are the backpatch to 18 and 19, which no committer has agreed to yet, and the wording of 0003, which is not shown.

Not checked: The 0003 text and the code of get_extension_control_directories() were not read, so the exact-match $system claim and the new long_desc wording rest on the commit messages. The function quoted in message #2 supports the 0001 claim.

## Prior attempts

None found before 2026-10-31.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does a committer agree to backpatch 0001 and 0002 to REL_18_STABLE and REL_19_STABLE?
2. What is the new long_desc text in 0003, and does it match the config.sgml wording from 0001?
3. Should the docs for 18 also say that pg_available_extensions silently skips an element like "$system/foo" while CREATE EXTENSION fails?

## Touch points

- **ABI / extension API:** 0001 changes the documented rule that extension authors follow for the "directory" control file parameter; the behavior stays the same.
- **new GUC:** No new GUC, but 0002 and 0003 change the documented description and long_desc of the existing extension_control_path.
- **backpatchability:** 0001 and 0002 ask for a backpatch to 18; 0003 changes a translatable string and stays on master.

## Who would commit it, and when

- Owner: silent. A committer who knows the extension loading code and extension_control_path would have to confirm the wording and the backpatch; none has replied in the visible thread.
- Priority: proceed; audience niche; review cost low. The patches are a few kilobytes of documentation that correct text contradicted by the code and by the repro in message #1. Two reviewers say v4 looks good; the audience is extension authors and administrators who set "directory" or extension_control_path.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
