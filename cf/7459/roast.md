# Design pre-review: plpgsql: detoast a returned record before keeping it in a non-atomic context

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7459](https://commitfest.postgresql.org/patch/7459/); [thread](https://www.postgresql.org/message-id/flat/CABUcsRRuMHXWqgiZ5wy%3D4OPbET0niFQEVY_d8VofeumTxjBJjw%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: low). No objection survived at levels 1a, 1b or 2. Message #1 states a reproducible failure on 14 through master and a fix that applies the detoast rule of 2efc924180f to one more assignment path. Confidence is low because only one message is visible and the code was not read.

Not checked: Without the code, nobody checked that ER_get_flat_size() inlines fields in the right memory context and leaves the record valid for takeover. Whether the CALL p(f()) path reaches the new step is also unchecked.

## Prior attempts

None found before 2026-10-15.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does the new step in exec_move_row_from_datum() also fix CALL p(f()), and does a test cover that case?
2. Do other paths that take over an expanded object in a non-atomic context, such as expanded arrays or OUT parameters, have the same gap?
3. What is the time cost when a non-atomic block assigns a record with large out-of-line fields many times, compared with unpatched master?
4. Do the TAP tests pass on master and on each back branch with the patch?

## Touch points

- **backpatchability:** The author asks for a back-patch to 14 through 19 and tags the entry Backport.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds one more detoast step in pl_exec.c that future takeover paths must also follow.
- **performance on the hot path:** Non-atomic r := f() now detoasts each out-of-line field of the returned record on assignment.

## Who would commit it, and when

- Owner: silent. A committer who knows PL/pgSQL's expanded-record handling and the non-atomic detoast rules of 2efc924180f would have to own it. None has spoken in the visible thread.
- Priority: needs champion; audience niche; review cost low. The patch is about 5 KiB, touches pl_exec.c and one isolation test, and fixes a reproducible "missing chunk number 0" error in non-atomic blocks. The entry has no reviewer, and a back-patch needs a committer who knows PL/pgSQL.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
