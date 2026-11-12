# Design pre-review: Deadlock detector fails to activate on a hot standby replica

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7372](https://commitfest.postgresql.org/patch/7372/); [thread](https://www.postgresql.org/message-id/flat/44c24dcf-5710-410f-b1b6-d10b315f3d51%40postgrespro.ru), 43 of 43 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). Nothing that survived questions the goal or the core design: this is a fix for a reproduced recovery stall, and the wait-loop approach was settled in review. The one surviving objection, maintenance-fit-2, is fixable: the stable-branch patches stop at v6.1 and lack the lock-conflict fix and the timeouts_armed flag from v10. What remains is a committer's call on the backpatch shape and on the commit order against the overlapping patches.

Not checked: Without the code, nobody can confirm here that the new inner loops cannot miss an unpin or a lock release, or that v10 applies to the stable branches. The claim that the v10 tests exercise the fixed path is also unchecked.

## Prior attempts

- [Revert "Get rid of the dedicated latch for signaling the startup process".](https://www.postgresql.org/message-id/flat/d8c0c608-021b-3c73-fffd-3240829ee986@oss.nttdata.com) (related; Reverted, 2020-12-17). "Because it broke the assumption that the startup process waiting for the recovery conflict on buffer pin should be waken up only by buffer unpin or the timeout enabled in ResolveRecoveryConflictWithBufferPin()." Different now: P1 was reverted because extra wakeups broke the buffer pin wait in ResolveRecoveryConflictWithBufferPin(). This proposal accepts that unrelated wakeups happen, as message #3 notes, and makes the wait loop recheck and keep waiting, so it no longer relies on that assumption.

## Objections a reviewer is likely to raise

### 1. Backpatches lag the master fix and miss the lock case

A committer pushes v10 to master and the existing v6.1 patches to REL_14 through REL_18. Stable branches keep the ResolveRecoveryConflictWithLock hang, and their buffer-pin loop differs from master's. Later fixes in this area, such as CF 6445 or the patch in message #24, then need separate versions for each branch.

Message #19 posted the stable-branch patches at v6.1. Later versions added the refcount recheck (messages #32-#34), the lock-conflict fix and the timeouts_armed flag (message #39), and none of these were backported. Other patches touching ResolveRecoveryConflictWithBufferPin are waiting on this one (messages #26, #31), and the bufmgr.c/standby.c refactor is deferred (message #29), so the branches will keep diverging.

**Question:** Will v10, including the lock-conflict fix, be backported to every stable branch, and in what order will it land relative to CF 6445 and the related ResolveRecoveryConflictWithBufferPin patches?

<sub>fixable, level 2, backpatchability; messages [#19](https://www.postgresql.org/message-id/7b1e8353-3d0a-48a1-bac6-2ac6ca91fbde%40postgrespro.ru), [#24](https://www.postgresql.org/message-id/CACSdjfMBDtxiFyA3eEVNJRQi1rufPGEjXOtAQ3qQkVc-P%3D187w%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CABPTF7WLTqtAL8DthdoMk5tKiVa22C4Urkbrcw78186eayracQ%40mail.gmail.com), [#29](https://www.postgresql.org/message-id/CAG5KqAbuFFgVfNoDHhiKGsP1twSjBx_VBpyfMsOY9VerPFVfnA%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/CACSdjfPCKNx9txxs5MGEhVUNC4H8nXkANgYAVR0%2Bb8kef_1P%2Bg%40mail.gmail.com), [#39](https://www.postgresql.org/message-id/CABPTF7XOitC_y0FNnm1hPKC%3DhZ6zL0kGU1W8W0MtS6iGOQX9DA%40mail.gmail.com), [#42](https://www.postgresql.org/message-id/CAGRkXqRswzdS%2BWAQO_VoKi4zk5qso1Cop7x_2t53aq1QUuFEyg%40mail.gmail.com); `maintenance-fit-2`</sub>

## Questions for the author

1. Will v10, including the ResolveRecoveryConflictWithLock fix and the timeouts_armed flag, be posted for each stable branch from REL_14_STABLE to REL_18_STABLE?
2. In what order should this patch, CF 6445 and the other patches that touch ResolveRecoveryConflictWithBufferPin be committed?
3. Do the v10 TAP tests fail reliably on an unpatched server, given that they check the mechanism and not the hang?
4. Do the stable-branch versions only add functions to bufmgr.h, with no change to an existing exported struct or signature?
5. Should the patch add a comment in timeout.c saying that handle_sig_alarm can set the latch when no active timeout fired?

## Touch points

- **WAL / replication / standby:** The patch changes how the startup process waits for buffer pins and locks during hot standby replay.
- **ABI / extension API:** The patch adds exported functions to bufmgr.h, and the entry targets stable branches.
- **backpatchability:** The fix targets REL_14_STABLE through REL_18_STABLE, but stable-branch patches exist only up to v6.1.
- **maintenance burden ("who maintains this in 10 years"):** Five stable branches carry the new wait loop, and other patches touch ResolveRecoveryConflictWithBufferPin.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao, who committed the P1 revert in this code, reviewed in message #3. The brief says no committer has replied since June, so someone who knows the recovery conflict and buffer pin wait code must still agree to take it.
- Priority: proceed; audience a sizeable minority; review cost medium. The bug stalls replay on hot standbys, most severely with max_standby_streaming_delay = -1, and several people reproduced it. The patch touches three files but must be carried on five stable branches, and the stable-branch versions need a refresh to v10 first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
