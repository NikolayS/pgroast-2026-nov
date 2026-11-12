# Design pre-review: [PATCH v1 0/1] Avoid carrying self lock-only xmax to updated tuple

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7253](https://commitfest.postgresql.org/patch/7253/); [thread](https://www.postgresql.org/message-id/flat/20260903092738.2352692-1-chee.wooson%40gmail.com), 6 of 6 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). Only standards-compat-1 survived, and it is a fixable level 2 point: after ON CONFLICT DO UPDATE the new row version shows xmax = 0 from SQL. No surviving objection questions the goal or the design. The open matter is whether the same-XID exception is correct under concurrency, and only a committer who knows heap tuple locking can sign that. Confidence is low because no code was read and no one in the visible thread reviewed correctness.

Not checked: The code was not read, so this review cannot confirm that the exception excludes MultiXacts and other subtransactions as described, or that the old tuple's xmax handling is unchanged. No isolation or concurrency stress test covers the change.

## Prior attempts

None found before 2026-09-03.

## Objections a reviewer is likely to raise

### 1. Updated rows from ON CONFLICT DO UPDATE now show xmax = 0

An application runs INSERT ... ON CONFLICT DO UPDATE ... RETURNING (xmax = 0) AS inserted, a widely copied idiom, to tell inserts from updates. After upgrade, every updated row returns inserted = true. The application miscounts or skips its update branch, and nobody notices until the data is wrong.

Message #5, test 3, shows that the patched server leaves xmax = 0 on the new version after ON CONFLICT DO UPDATE, where it was the updating XID before. v2 changes the existing ON CONFLICT expected output in update.out, so the change is visible from SQL as well as through pageinspect. The idiom is undocumented, but a committer would likely want it named in the release notes, with a pointer to a supported way to tell inserts from updates. The Bugfix tag also suggests a backpatch, and making this change in a minor release would be harder to defend.

**Question:** Will the patch stay master-only with a release-note entry for applications that read xmax after ON CONFLICT DO UPDATE, and what replacement will that entry point to?

<sub>fixable, level 2, backpatchability; messages [#4](https://www.postgresql.org/message-id/CAMfungPiUSxdTR%2BHNRhV0sVXTQgguFtuWgVrGBFKSY8dpgyC3w%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CADMs5OPxgrppCUfx6JsZkOZhhQ9XcpMEpzp_UN6BP%2BKgyrb3nw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Is the patch for master only, and will the release notes name the xmax = 0 change after ON CONFLICT DO UPDATE and point to a supported replacement?
2. Which isolation test shows that a concurrent FOR KEY SHARE locker and a lock from a different subtransaction still keep their xmax on the new tuple version?
3. Does the same-XID exception hold when the first lock is FOR SHARE, FOR KEY SHARE or FOR NO KEY UPDATE instead of FOR UPDATE?
4. Does v2 keep the README.tuplock section that states the invariant?
5. How did the 2019 discussion cited in message #1 end, and why was this change not made then?

## Touch points

- **on-disk format / pg_upgrade:** heap_update() writes xmax = 0 with HEAP_XMAX_INVALID on the new tuple version in the same-XID case; the state already exists on heap pages.
- **WAL / replication / standby:** The update WAL record carries the new tuple's xmax and infomask, so standbys replay the changed value.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Updated rows from ON CONFLICT DO UPDATE now show xmax = 0
- **backpatchability:** The entry is tagged Bugfix but targets version 20, and the thread does not say whether a backpatch is intended.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a same-XID special case to tuple-lock carrying in heapam.c, backed by an invariant in README.tuplock.
- **performance on the hot path:** The change is inside heap_update(), which every UPDATE runs; the measured gain covers one lock-and-update loop only.

## Who would commit it, and when

- Owner: silent. Someone who knows heap tuple locking, MultiXacts and the xmax infomask rules in heapam.c would have to own it. No such person has reviewed the same-XID exception in the visible thread.
- Priority: needs champion; audience niche; review cost medium. The patch is small and adds no user-facing feature, and the measured gain covers lock-and-update loops in subtransactions. Review cost comes from the tuple-locking reasoning, and no one with that background has commented yet.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
