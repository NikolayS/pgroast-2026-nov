# Design pre-review: Don't close relation files when the visibility map is extended

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7524](https://commitfest.postgresql.org/patch/7524/); [thread](https://www.postgresql.org/message-id/flat/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). No surviving objection questions the goal: message #1 reports a production stall with numbers and message #2 reproduces the cost independently. The remaining points are fixable design questions. maintenance-fit-1 asks for the rule that makes the size-only message safe, and replication-ondisk-1 asks whether the new message ID can reach WAL and which branches are targeted. Both need a committer who knows the smgr and sinval code, and the thread has only two messages.

Not checked: Without the code, nobody could check whether the new message stays out of the transactional invalidation queue, or whether any path relied on vm_extend() closing files. The reason for the standbydesc.c change is also unverified.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. Second smgr invalidation kind that callers must choose correctly

A later patch that grows, truncates or rewrites a fork copies the vm_extend() call and sends the fork-size-only message. Backends then keep descriptors for segments that were removed, or keep other smgr state that a close used to reset. The result is a rare wrong-size or stale-file bug found in the field, not a test failure.

Today one smgr invalidation closes everything, so a caller cannot pick the wrong strength. The patch adds a weaker message that is safe only when no segment disappears and no other per-relation smgr state needs a reset, and the proposal does not state that rule anywhere a future author would see it. In [P1], removing an smgrclose() also skipped resetting smgr_cached_nblocks and set up a convention that smgrclose() may be left out in one special case. The same kind of convention appears here.

Precedent: [Remove duplicated smgrclose call](https://www.postgresql.org/message-id/flat/CABBtG=cDTCBDCBK7McSy6bJR3s5xUTOg0vSFfuW8oLdUYyCscA@mail.gmail.com) (Returned with feedback, 2025-08-05)

**Question:** What invariant makes the size-only message safe for vm_extend(), and where is it documented or asserted so a later caller cannot use it after truncation?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com); `maintenance-fit-1`</sub>

### 2. New invalidation message type is unsafe to back-patch across minor versions

A user runs a primary on the patched minor release and a hot standby on the previous one. The primary writes a commit or XLOG_INVALIDATIONS record that carries the new message ID. The older standby has no handler for that ID, so replay or logical decoding errors out, and the user finds out when the standby stops applying WAL.

The patch touches standbydesc.c, which describes invalidation messages carried in WAL, so the new ID may be written to WAL records. Message #1 notes the problem exists in every branch from 14 to master. The proposal does not say whether it targets those branches or master only. It also does not say whether the new ID always stays out of WAL; vm_extend() sends its invalidation directly, but the material does not confirm the new ID never enters the transactional queue.

**Question:** Can the new message ID ever be written to WAL, and if so, is the patch meant for master only or for back-branches too?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com); `replication-ondisk-1`</sub>

### 3. Point fix for one source of a general reopen cost

After this lands, the same encrypted-filesystem user still sees the inserts stall when DDL, truncation or other relcache and smgr invalidations hit a large table. Each source then gets its own special message, and the number of invalidation variants grows release by release.

Message #2 says DDL invalidations cause the same load and asks whether Postgres should optimize for costly open() at all. That question has no answer in the thread. A committer would likely ask whether the cost belongs at the smgr or fd layer, such as keeping descriptors across an invalidation when segments still exist, so that every invalidation source gains instead of one.

**Question:** Why is a vm_extend()-specific message better than making smgr invalidation keep open segments in general, and which other sources would still cause mass reopens?

<sub>fixable, level 1b, design; messages [#2](https://www.postgresql.org/message-id/CAKZiRmxmPL_7grAoEF6mxuyWXUi8BCnZZpaEaF05AX2NDfFP0g%40mail.gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. What invariant makes it safe for other backends to keep their segment files open after vm_extend(), and where does the patch state or assert it?
2. Can the new message ID ever be written to a WAL record, such as a commit or XLOG_INVALIDATIONS record?
3. Is the patch meant for master only, or for back-branches down to 14?
4. Why is a vm_extend()-specific message better than making the smgr invalidation keep existing segments open in general, and which other sources would still cause mass reopens?
5. Does a benchmark on ext4 or XFS show no regression from the new message?

## Touch points

- **WAL / replication / standby:** The patch touches standbydesc.c, which describes invalidation messages carried in WAL records.
- **ABI / extension API:** The patch adds a message type in sinval.h and changes declarations in smgr.h and inval.h.
- **backpatchability:** Message #1 reports the problem on 17 and names branches 14 to master, but states no target branch.
- **maintenance burden ("who maintains this in 10 years"):** A second, weaker smgr invalidation kind exists that later callers must choose correctly.
- **performance on the hot path:** The patch changes what every backend does on receiving the invalidation that vm_extend() sends during VACUUM and pruning.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** Message #2 asks whether Postgres should optimize for filesystems with costly open() at all.

## Who would commit it, and when

- Owner: silent. Someone who knows the smgr and md layers and shared invalidation handling, including how invalidation messages reach WAL and standbys. No such committer has spoken in the two visible messages.
- Priority: proceed; audience niche; review cost medium. The measured gain is large but shows on filesystems with costly open(), large tables and many writers; other users only lose unneeded reopens. The patch is small, about 10 KiB over seven files, but it adds a permanent invalidation kind in smgr and sinval code that needs expert review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
