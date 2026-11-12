# Design pre-review: Remove redundant MultiXactIdIsRunning() check in HeapTupleSatisfiesUpdate()

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7418](https://commitfest.postgresql.org/patch/7418/); [thread](https://www.postgresql.org/message-id/flat/7201E833-0100-4BDE-9161-45B9FFA4F458%40icloud.com), 4 of 4 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because the retained comment states the dependency on the first check and multixact membership is a documented invariant. The patch removes a branch that message #1 argues cannot be reached, and one reviewer agreed in message #3. Confidence is medium because the code was not read and no committer has confirmed the reasoning.

Not checked: The code was not read, so the claim that nothing between the two checks can change the result rests on message #1 and message #3. The lock held across both checks could not be checked.

## Prior attempts

- [Remove redundant NULL check in clause_selectivity_ext()](https://www.postgresql.org/message-id/flat/016e33b7-2830-4300-bc89-e7ce9e613bad@tantorlabs.com) (similar idea; Rejected, 2024-09-30). "Certainly, such an assumption could be argued to be a bug they should fix; but I'm disinclined to make them jump through that hoop for a vanishingly small performance improvement." ([message](https://www.postgresql.org/message-id/1373706.1724079757%40sss.pgh.pa.us)) Different now: P2 removed a defensive NULL check in a function that extensions call, so outside callers could depend on it. This patch removes a branch inside HeapTupleSatisfiesUpdate() that no caller can reach, because multixact membership is immutable.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Do both MultiXactIdIsRunning() calls run under the same buffer lock, and does the argument depend on that?
2. Do other visibility routines, such as HeapTupleSatisfiesMVCC, have the same double check, and should this patch cover them?
3. Is there a measurement of the saved member lookup and proc array scan, or is the patch justified by readability alone?

## Touch points

- **maintenance burden ("who maintains this in 10 years"):** The patch edits HeapTupleSatisfiesUpdate() in heapam_visibility.c, where setting HEAP_XMAX_INVALID now relies on the first MultiXactIdIsRunning() check alone.
- **performance on the hot path:** The patch removes a multixact member lookup and proc array scan from the aborted-updater path of a tuple visibility routine.

## Who would commit it, and when

- Owner: silent. Someone who knows heap tuple visibility and multixact handling would have to own it. No committer has replied in the visible thread.
- Priority: needs champion; audience niche; review cost low. The patch is about 2 KiB in one file, tagged Refactoring Only, with no user-visible change and no measured gain. It needs a committer who knows multixact visibility to confirm the reasoning, and none has replied.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
