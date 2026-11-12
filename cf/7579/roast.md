# Design pre-review: Fix ReadStream handling when heap rescan strategy changes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7579](https://commitfest.postgresql.org/patch/7579/); [thread](https://www.postgresql.org/message-id/flat/aumB7BHGWewR4x%2FS%40bdtpg), 6 of 6 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived the critic; the only one raised, maintenance-fit-1, was refuted because the material does not show a field-by-field copy in heapam.c. The proposal is a bug fix for b7b0f3f27241 with a stated defect, a reachable case in core (message #3) and no new user-visible surface. Two reviewers support a separate fix. The result is provisional because the code was not read.

Not checked: Without the code, the read_stream.c change, its ABI effect on back branches, and the place where the statistics carry-over lives could not be checked. Stream recreation during parallel or sync scans also could not be checked.

## Prior attempts

None found before 2026-11-03.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which back branches does the fix target, and does the read_stream.c change alter any exported struct or function that those branches must keep ABI-stable?
2. What changes in read_stream.c, given that message #1 says the fix uses only existing ReadStream APIs?
3. Does the statistics and maximum-capacity carry-over live inside read_stream.c, so that a new instrumentation field cannot be missed in heapam.c?
4. Is recreating the stream in initscan() safe for parallel scans and sync scans, where scan position state is shared or reported?
5. Should the 0002 TAP test, which starts its own cluster, go into the tree, or is there a cheaper way to cover the strategy switch?

## Touch points

- **ABI / extension API:** The patch changes read_stream.c, and the reported dangling pointer is reached by extensions that call table_rescan_set_params().
- **backpatchability:** The CommitFest entry targets stable branches, and message #1 gives easier backpatching as a reason for this approach.
- **maintenance burden ("who maintains this in 10 years"):** The recreate path carries instrumentation statistics and maximum capacity from the old stream to the new one.
- **performance on the hot path:** initscan() runs on every heap rescan, though the stream is recreated only when strategy use switches.

## Who would commit it, and when

- Owner: silent. A committer who knows the ReadStream and AIO code and the heap scan setup in heapam.c would have to own it. The visible thread and the CommitFest entry do not identify any participant as a committer.
- Priority: proceed; audience niche; review cost low. The fix is a small patch to two files that corrects a defect from b7b0f3f27241. Core reaches only the stale NULL strategy case with extra buffer pollution (message #3); the freed-pointer case needs an extension.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
