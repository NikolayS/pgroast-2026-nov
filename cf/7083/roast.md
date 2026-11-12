# Design pre-review: Fix write skew under SERIALIZABLE for Tid Scan & Tid Range Scan

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7083](https://commitfest.postgresql.org/patch/7083/); [thread](https://www.postgresql.org/message-id/flat/CA%2BCOZaBo%2BZpKgMvxcdACUjNtdYipe9Em06iX5KHLTVaTmFibiw%40mail.gmail.com), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The TID range scan fix (v5-0001) follows the existing seqscan rule and no surviving objection is against it. The TID scan fix (v5-0002) brings back the relation-level locks that cb5b28613d5 removed, and message #14 does not say why a narrower lock is not enough (prior-attempts-1). The entry targets stable branches without saying whether that includes v5-0002 (maintenance-fit-2). Confidence is medium because the v5 patches and their commit messages were not shown.

Not checked: The v5 patches, their commit messages and the README-SSI changes were not shown, so the exact lock condition in v5-0002 is taken from messages #13 and #14. Whether a lock taken only for probes that find no tuple closes the gap in message #11 cannot be judged without the code.

## Prior attempts

None found before 2026-07-27.

## Objections a reviewer is likely to raise

### 1. TID scan relation lock reverses cb5b28613d5 without addressing its reason

An application runs a SERIALIZABLE cursor loop with UPDATE ... WHERE CURRENT OF on a table that other sessions write to. After a minor upgrade with v5-0002, each loop takes a relation-level SIREAD lock, so any concurrent write to that table can abort it. The application then sees new serialization failures with no change on its side.

Message #13 reports that cb5b28613d5 added SO_TYPE_TIDSCAN in 2020 to avoid relation-level locks for TID scans, and v5-0002 brings those locks back. Message #14 cites false positives being allowed by design, but does not say why the reasoning of that commit was wrong. The bug in message #11 needs a lock only when a probe finds no tuple, while v5-0002 locks the relation for every TID scan. The CF entry targets stable branches, so the extra aborts would arrive in minor releases.

**Question:** Why does v5-0002 lock the relation for TID probes that find a visible tuple, and for WHERE CURRENT OF, when cb5b28613d5 deliberately avoided that?

<sub>fixable, level 2, prior rejection; messages [#11](https://www.postgresql.org/message-id/CAN4CZFOfQL_nOs2ukYf-CkcdP6EOQHu0dx2A%2Ba90F%2BKDqajLKg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAJ7c6TMEJ7sZhASiK5_VjnO6yfr-yawZ_FFzFYBVRFQWNPm7ig%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAN4CZFMs1W11eBpvo5bJsGaXoEBKkdcgZiBW9VzWTmGkD3zW5w%40mail.gmail.com), [#14](https://www.postgresql.org/message-id/CAJ7c6TMCQmm-7akUPPE0GoAwYdeN5E%3D%3DKU87%3DE8g9yjArVHXMQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 2. Back-patching a contested TID scan change adds aborts in minor releases

A site on Postgres 14 to 19 applies a minor update. Its SERIALIZABLE workload, which uses ctid lookups or cursor updates, starts to fail with serialization errors at a higher rate. Nothing in the release notes points the site to the cause, so it finds out from application retries and errors.

The CommitFest entry targets stable branches, and both v5 patches sit in one entry. v5-0001 for TID range scans has support in message #7, but v5-0002 is disputed in message #13 and partly undoes an earlier deliberate design choice. Changing locking behaviour that affects ordinary cursor loops is a larger change than the range-scan gap, and the proposal does not separate the two for back-patching.

**Question:** Is v5-0002 meant for stable branches, and if so, what evidence shows that the added serialization failures are acceptable in a minor release?

<sub>fixable, level 1b, backpatchability; messages [#7](https://www.postgresql.org/message-id/3670DAB7-E3BE-48AC-8DF9-884509775F78%40yandex-team.ru), [#12](https://www.postgresql.org/message-id/CAJ7c6TMEJ7sZhASiK5_VjnO6yfr-yawZ_FFzFYBVRFQWNPm7ig%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAN4CZFMs1W11eBpvo5bJsGaXoEBKkdcgZiBW9VzWTmGkD3zW5w%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Does v5-0002 take the relation lock for every TID scan, or only when a TID probe finds no tuple?
2. Why is a lock taken only for probes that find no tuple not enough, given that cb5b28613d5 added SO_TYPE_TIDSCAN to avoid relation-level locks?
3. Is v5-0002 meant for stable branches, or only v5-0001?
4. How much do serialization failures rise for a SERIALIZABLE cursor loop with UPDATE ... WHERE CURRENT OF on a concurrently written table with v5-0002 applied?
5. Can v5-0001 be committed and back-patched alone while the TID scan lock scope is still under discussion?

## Touch points

- **backpatchability:** The entry targets stable branches, so new serialization failures for TID scans and WHERE CURRENT OF would arrive in minor releases.
- **performance on the hot path:** v5-0002 takes a relation-level SIREAD lock for TID scans under SERIALIZABLE, which raises the abort rate for cursor loops and ctid updates.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund asked for the SO_TYPE note and the bitmap scan tests, and both were added. The owner needs to know SSI predicate locking and the heap scan flags, and nobody has yet settled the lock scope dispute on v5-0002.
- Priority: fix design first; audience niche; review cost low. The bug is a reproducible SERIALIZABLE anomaly, and the change is small and limited to heapam.c plus isolation tests. v5-0001 looks ready for a committer, but the lock scope of v5-0002 and its back-patch status need agreement first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
