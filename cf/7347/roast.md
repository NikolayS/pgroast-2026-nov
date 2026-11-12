# Design pre-review: Temporary slot leak when creation fails in a subtransaction

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7347](https://commitfest.postgresql.org/patch/7347/); [thread](https://www.postgresql.org/message-id/flat/CALj2ACXDsQhq0MFuyRfZ0BJT6F93CNQmG5iv%2BwDzKp%2BYyOGD%2BA%40mail.gmail.com), 7 of 7 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is confirmed in message #2 and the fix changes nothing on disk or in shared memory. The one surviving objection, already-exists-1, is a fixable design point: on HEAD the backend-local variable would sit beside RS_EPHEMERAL, and the author's reason against ephemeral-first creation applies to back branches only. That choice, and a back-patch to 14, need a committer who knows the slot code.

Not checked: Without the code, two points are unchecked: whether a slot whose creation finished before a later error in the same subtransaction keeps its documented behaviour, and whether walsender slot creation outside a transaction is unaffected. Only 3 of 7 messages were shown.

## Prior attempts

None found before 2026-09-25.

## Objections a reviewer is likely to raise

### 1. Second "creation in progress" mechanism next to the existing ephemeral state

A later patch adds a slot creation path, such as a new sync or creation function, and uses only one of the two mechanisms. Nothing fails at once. The leak comes back in that path and is found only when someone hits a caught error during creation.

Postgres already has RS_EPHEMERAL, which drops a persistent logical slot on release if its creation fails. The patch adds MyReplicationSlotCreating for temporary slots, and message #6 proposes extending it to persistent physical slots, so two mechanisms would do the same job on HEAD. The author calls ephemeral-first invasive and not back-patchable, but that argument covers back branches only. In message #7, shveta asks for the ephemeral state on HEAD, and the thread has not settled this.

**Question:** On HEAD, what stops temporary and physical slots from starting as RS_EPHEMERAL, so that the backend-local variable is needed only on back branches?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CALj2ACXDsQhq0MFuyRfZ0BJT6F93CNQmG5iv%2BwDzKp%2BYyOGD%2BA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CALj2ACV%2BOzVikz4dwgWi399AHas3RneVsWfy-sF43bCf65CprA%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAJpy0uAnNgCaOpQcum2uqdq-4in_k7cQc0vqzVbyufeMfpwGCA%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. On HEAD, what stops temporary and physical slots from starting as RS_EPHEMERAL, so that the backend-local variable is needed only on back branches?
2. In which order are this patch and the v19 patch from the other thread to be committed, on HEAD and on each back branch?
3. Is the CreateDecodingContext() check for an invalid confirmed_flush part of this patch, and on which branches?
4. How does the patch treat a walsender that creates a temporary slot outside a transaction?
5. Will the same variable also cover failed creation of a persistent physical slot, or is that a separate patch?

## Touch points

- **WAL / replication / standby:** The patch changes when a temporary replication slot is dropped, in src/backend/replication/slot.c.
- **wire protocol:** protocol.sgml changes the documented behaviour of CREATE_REPLICATION_SLOT ... TEMPORARY; no message format changes.
- **backpatchability:** The commit message asks for a back-patch to 14, and the patch applies only on top of an unmerged patch.
- **maintenance burden ("who maintains this in 10 years"):** HEAD would carry a backend-local creation variable beside the existing RS_EPHEMERAL state.

## Who would commit it, and when

- Owner: silent. A committer who knows replication slot lifetime, logical decoding and subtransaction abort handling. The visible messages show reviewers, and no such committer speaking.
- Priority: proceed; audience niche; review cost medium. The trigger is narrow, a failed temporary slot creation caught in a subtransaction, but the result is retained WAL and partial decoding, and the fix has no cost outside that path. Review needs care in slot.c and postgres.c on six branches, and the patch needs a rebase.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
