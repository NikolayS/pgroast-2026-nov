# Design pre-review: WAL record CRC calculated incorrectly because of underlying buffer modification

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7532](https://commitfest.postgresql.org/patch/7532/); [thread](https://www.postgresql.org/message-id/flat/055bb33c-bccc-bc1d-c2f8-8598534448ac%40gmail.com), 11 of 11 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: low). This is a real bug with buildfarm failures and a reproducer, so levels 1a and 1b pass. The visible material holds no fix: the only attachment is a debugging assertion, and message #10 rules out the one candidate. The surviving objections, replication-ondisk-2 and maintenance-fit-1, are fixable design points about the back-branch plan and an unenforced REGBUF_NO_CHANGE rule. The second CommitFest thread and any newer patch are not shown.

Not checked: The fix itself could not be judged: the only visible patch is the debugging assertion, and the second thread attached to the CommitFest entry is not shown. The back-branch approach and the lock cost are unknown.

## Prior attempts

None found before 2024-05-10.

## Objections a reviewer is likely to raise

### 1. No back-branch fix that keeps hash WAL records replayable

An operator applies a minor release to the primary before the standby, as the upgrade notes advise against but as rolling upgrades often do. If the fix changes which blocks a hash record registers or how it registers them, the older standby fails to replay, or replays differently, and the operator finds out when replication stops.

Message #6 reproduces the bug on a 2018 commit, so every supported branch needs a fix. Message #5 notes that the back branches lack REGBUF_NO_CHANGE, and message #7 offers only two untested ideas. Message #10 rules out the one candidate fix, and the fixes Jeff Davis suggests (exclusive locking, or copying the page as XLogSaveBufferForHint() does) may change hash record contents or locking during replay. The CommitFest entry adds a second thread, but the material does not show it.

**Question:** Which fix goes into the back branches, and does it leave the hash WAL record format unchanged, so that standbys on older and newer minor releases replay the same records?

<sub>fixable, level 2, backpatchability; messages [#5](https://www.postgresql.org/message-id/CA%2BhUKGJEtb65B%2Bv2Vk25WbvvALH5DxL5Oye%3DDuhCVP8K1X3Obg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/35e2c2f8-f65a-f93a-ea23-957bb67a2d42%40gmail.com), [#7](https://www.postgresql.org/message-id/CA%2BhUKG%2B%3Dcb86CYa4W42z4wFBMwjQE2%3DO9RFC%2Bi4QZuCB%2Bd2p0A%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/a4e1196a77f154ccf8df486a632d998525a1becc.camel%40j-davis.com); `replication-ondisk-2`</sub>

### 2. A fix that keeps REGBUF_NO_CHANGE leaves its rules unenforced

A later hash or extension change registers a pinned, unlocked buffer with REGBUF_NO_CHANGE but without REGBUF_NO_IMAGE. Nothing fails at insert time. Months later a buildfarm member running wal_consistency_checking, or a user standby, logs "incorrect resource manager data checksum", and someone again has to trace it back.

Message #8 names an invariant (REGBUF_NO_CHANGE implies REGBUF_NO_IMAGE) that no code checks, and says _hash_freeovflpage() already breaks it. Message #10 rules out the special case in message #5. This bug survived from at least 2018 (message #6), so a silent rule is not enough protection. Messages #8 and #9 prefer removing the flag, but nobody has committed to that work.

**Question:** Will the fix remove REGBUF_NO_CHANGE, or add a check that fails loudly when a caller registers an unlocked buffer that can still produce an image?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#5](https://www.postgresql.org/message-id/CA%2BhUKGJEtb65B%2Bv2Vk25WbvvALH5DxL5Oye%3DDuhCVP8K1X3Obg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/35e2c2f8-f65a-f93a-ea23-957bb67a2d42%40gmail.com), [#8](https://www.postgresql.org/message-id/895723aba95f108207a744c799588ec3f2fef819.camel%40j-davis.com), [#9](https://www.postgresql.org/message-id/ZkavFZeGgPlaQ-W5%40paquier.xyz), [#10](https://www.postgresql.org/message-id/a4e1196a77f154ccf8df486a632d998525a1becc.camel%40j-davis.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Which fix does the current patch implement: an exclusive lock on the bucket page, a page copy as XLogSaveBufferForHint() does, or something else?
2. Which fix goes into the back branches, which lack REGBUF_NO_CHANGE, and does it leave the hash WAL record format unchanged?
3. Will the fix remove REGBUF_NO_CHANGE, or add a check that fails loudly when a caller registers an unlocked buffer that can still produce an image?
4. What does the chosen fix cost in hashbucketcleanup() during vacuum of a hash index under concurrent inserts?
5. Why does hashbucketcleanup() avoid the exclusive lock on the bucket page, and does the fix keep whatever that choice protects?

## Touch points

- **WAL / replication / standby:** The bug is a wrong WAL record CRC, and any fix changes how hash code registers buffers for WAL.
- **ABI / extension API:** Removing or restricting REGBUF_NO_CHANGE changes the XLogRegisterBuffer() flags that extensions can use.
- **backpatchability:** Message #6 reproduces the bug on a 2018 commit, and the back branches lack REGBUF_NO_CHANGE.
- **maintenance burden ("who maintains this in 10 years"):** No code checks that an unlocked registered buffer cannot produce a full-page image.
- **performance on the hot path:** An exclusive lock or page copy in hashbucketcleanup() adds cost to hash index vacuum, and nobody has measured it.

## Who would commit it, and when

- Owner: qualified committer engaged. Several committers who know WAL insertion replied in May 2024, and Jeff Davis proposed fix directions. The fix also needs someone who knows the hash index locking protocol, and message #11 leaves that question unanswered.
- Priority: proceed; audience niche; review cost medium. The bad CRC is shown only for hash indexes with wal_consistency_checking enabled, but it is a real bug that causes buildfarm failures. Review cost is a guess because no fix patch was read; the change is likely small but touches WAL insertion and hash locking on all branches.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
