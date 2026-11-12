# Design pre-review: Make the logical replication conflict messages more like each other

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7043](https://commitfest.postgresql.org/patch/7043/); [thread](https://www.postgresql.org/message-id/flat/CAHut%2BPuX_pPF6mU-UPRdQQToGgUPRpbOxOEFxPVC7B5Tevb-9A%40mail.gmail.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). Nothing that survived stops the proposal at levels 1a, 1b or 2. The one surviving objection, standards-compat-1, is a fixable wording point: two proposed reason lines are less exact than the detail lines they introduce. The proposal only rewords DETAIL text in errdetail_apply_conflict and adds no new interface. Confidence is low because no patch exists and the thread has one supporting reply.

Not checked: No patch exists, so the final strings, the translation impact and the test output changes could not be judged. The verdict rests on the strings suggested in message #1.

## Prior attempts

None found before 2026-05-18.

## Objections a reviewer is likely to raise

### 1. New reason lines can contradict the detail lines they introduce

An apply worker hits insert_exists on a plain unique index that has no constraint and logs "The row to insert violates a constraint". The DBA looks in pg_constraint, finds no matching entry, and loses time before reading the next line about the unique index. In update_origin_differs, the log says "modified by another origin" and then "It was modified locally".

Message #1 adds summary lines that are less exact than the detail lines below them. The insert_exists and update_exists checks look only at unique indexes, and elsewhere Postgres names the object exactly, as in "violates unique constraint". Calling a local change "another origin" is defensible only from the remote side, and the message does not say so. The proposal is about consistency, so a committer would likely ask that the summary lines first agree with the detail lines they introduce.

**Question:** Can the reason lines say exactly what was detected, such as a unique index and a local or other-origin change, without contradicting the detail line?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAHut%2BPuX_pPF6mU-UPRdQQToGgUPRpbOxOEFxPVC7B5Tevb-9A%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Can the reason line for insert_exists and update_exists name a unique index or unique key instead of "a constraint"?
2. How should the update_origin_differs and delete_origin_differs reason line read when the detail line says "It was modified locally"?
3. Does the logical replication conflicts page of the docs need changes to match the new text?
4. Which regression and TAP test expected outputs change, and by how much?

## Touch points

- **WAL / replication / standby:** The reworded messages come from the logical replication apply worker; replication behaviour does not change.
- **backpatchability:** Changed message text needs new translations, so the change fits a new major version only.
- **maintenance burden ("who maintains this in 10 years"):** One-time translator and expected-output updates; later conflict types must follow the shared pattern.

## Who would commit it, and when

- Owner: silent. Someone who knows the logical replication apply worker and conflict detection code, and the error message style guide. No committer has spoken in the visible thread.
- Priority: needs champion; audience niche; review cost low. The change rewords strings in one function and benefits people who read apply-worker conflict logs. It is waiting for agreement from someone who knows the area, because the author will not write a patch without it and only one +1 has arrived.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
