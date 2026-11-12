# Design pre-review: Release replication slot on error in SQL-callable slot functions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7097](https://commitfest.postgresql.org/patch/7097/); [thread](https://www.postgresql.org/message-id/flat/CAHg%2BQDeuf9tCq3ce%3DkgFMJP0m%3DPZC%2Bwi6B%3DyS%2B7V0vNXjLS31w%40mail.gmail.com), 90 of 90 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is reproducible with plain SQL and the goal is not in dispute. One fixable level 2 objection survived, standards-compat-1: releasing the slot on subtransaction abort changes what extensions see in minor releases back to PG14, and the thread reviewed only the commit side. That is a back-patch risk judgement for a committer who knows the slot code. It does not stop the idea.

Not checked: The material shows commit messages for v1 and v5 only, so the v19 callback design and its claim that release cannot raise an error during subtransaction abort were not checked. The thread does not say whether v19 was committed.

## Prior attempts

None found before 2026-05-09.

## Objections a reviewer is likely to raise

### 1. Release on subtransaction abort changes extension-visible slot state in minor releases

An extension acquires a slot inside BeginInternalSubTransaction(), catches an error with PG_CATCH, calls RollbackAndReleaseCurrentSubTransaction() and then calls ReplicationSlotRelease(). After the minor update the rollback has already cleared MyReplicationSlot, so the extension's release call fails an assertion or dereferences a NULL slot. The user finds out when the backend crashes on the first decoding error after the update.

The thread studied the commit side for extensions in detail and removed the handoff and WARNING to avoid minor-release changes (messages #70, #73, #81). The abort side gets no such review, but it is the part that now runs inside AbortSubTransaction() on every supported branch back to PG14. Existing extension cleanup code was written for a world where slot state survives a subtransaction rollback. The v19 commit message is not in the material, so it is unknown whether release notes will warn extension authors.

**Question:** Was extension code that releases the slot after rolling back its own subtransaction checked, and will the back-branch commit message or release notes tell extension authors about the new abort behavior?

<sub>fixable, level 2, backpatchability; messages [#70](https://www.postgresql.org/message-id/CALj2ACUVTXdtGSdH7ez7anCB1cT8o78KB9RJ5c1uutRd0OsgsQ%40mail.gmail.com), [#73](https://www.postgresql.org/message-id/CAD21AoB-dkBCo06isKxeF-W4Vah8YnoGVbOBOs1SysoSQRr45w%40mail.gmail.com), [#81](https://www.postgresql.org/message-id/CALj2ACUBJ53hY8EOUJnWA7m1GzBP_CoPo9N6e01x5%2Bq9GPQoXQ%40mail.gmail.com), [#86](https://www.postgresql.org/message-id/CAD21AoDT_ypWw%2BGpQ377Dat4w%3Dj7iqhqC1m-X2XYDxtXjYeBtA%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Was extension code that calls ReplicationSlotRelease() after rolling back its own subtransaction checked against the new abort behavior?
2. Will the back-branch commit message or release notes tell extension authors that a subtransaction abort now releases the slot?
3. Should the abort-time release go to all branches back to PG14, or should back branches get a narrower fix?
4. How does a slot that an extension holds across transactions behave after this change?
5. Why is a subtransaction-end callback preferred over ResourceOwner, as message #55 suggested, at least on HEAD?

## Touch points

- **WAL / replication / standby:** The patch changes when a backend releases a replication slot, which affects WAL retention by an active slot.
- **ABI / extension API:** A slot that an extension acquires in a subtransaction is now released when that subtransaction aborts.
- **backpatchability:** The fix is meant for all supported branches back to PG14 and changes slot behavior on subtransaction abort there.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a subtransaction-end callback and a recorded subtransaction id that slot code must keep correct.
- **performance on the hot path:** The new callback runs at every subtransaction end, and the review notes put the cost at one variable comparison.

## Who would commit it, and when

- Owner: qualified committer engaged. Sawada reviewed the design and said in message #86 that he planned to push. The owner needs to know the replication slot code and subtransaction abort processing.
- Priority: proceed; audience niche; review cost medium. Only sessions that catch a slot function error in a subtransaction hit the bug, but the result is an assertion failure or a slot stuck active that holds back WAL removal and vacuum. The v19 patch was not read; early versions are small and touch slot code only.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
