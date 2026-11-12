# Design pre-review: Don't close relation files when the visibility map is extended

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7524](https://commitfest.postgresql.org/patch/7524/) (Needs review); authors: Bryan Green (lispyone)
- Thread: [Don't close relation files when the visibility map is extended](https://www.postgresql.org/message-id/flat/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com), started 2026-10-26
- Messages read: 2 of 2
- Patch prose read: v1-0001-visibility-extends-should-not-cause-file-opens.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

No surviving objection questions the goal: message #1 reports a production stall with numbers and message #2 reproduces the cost independently. The remaining points are fixable design questions. maintenance-fit-1 asks for the rule that makes the size-only message safe, and replication-ondisk-1 asks whether the new message ID can reach WAL and which branches are targeted. Both need a committer who knows the smgr and sinval code, and the thread has only two messages.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody could check whether the new message stays out of the transactional invalidation queue, or whether any path relied on vm_extend() closing files. The reason for the standbydesc.c change is also unverified.

## The proposal, restated

**Problem.** When vm_extend() grows the visibility map it sends an smgr invalidation, and every backend with the relation open closes all its segment files and reopens them later. On filesystems where open() is costly, such as encryption layers, this stalls inserts on large tables while VACUUM or on-access pruning extends the map.

**Proposed design.** The patch adds a new shared invalidation message that only resets the cached fork sizes, and vm_extend() sends it instead of the full smgr invalidation. Backends keep their files open and see new map pages when they do today. The author rejected dropping the invalidation, because a backend that does not prune the pages keeps a stale map size until VACUUM finishes.

**User-visible surface.**
- New shared invalidation message type in sinval.h.
- New or changed C functions in inval.h and smgr.h for sending and handling the fork-size reset.
- New invalidation message description in standbydesc.c, so pg_waldump output for standby or commit records can change.

**What the author claims.**
- None of the file reopens caused by vm_extend() are needed; other backends only need to learn the new map size.
- On eCryptfs with about 2,700 segments and 300 inserting clients, VACUUM caused about 10 million opens before and under 1 million after.
- p99 insert latency during VACUUM dropped from about 350ms to under 20ms, and the extra system CPU mostly went away.
- The same invalidation exists in every branch from 14 to master; check-world passes.

**Thread state.** Jakub Wartak traced the invalidation to commit b4e074172768, reproduced high system CPU on encrypted filesystems, and asked whether Postgres should optimize for costly open() at all, since DDL invalidations cause the same load. The author's question about why the full invalidation is needed has no answer yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Second smgr invalidation kind that callers must choose correctly

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch that grows, truncates or rewrites a fork copies the vm_extend() call and sends the fork-size-only message. Backends then keep descriptors for segments that were removed, or keep other smgr state that a close used to reset. The result is a rare wrong-size or stale-file bug found in the field, not a test failure.

Today one smgr invalidation closes everything, so a caller cannot pick the wrong strength. The patch adds a weaker message that is safe only when no segment disappears and no other per-relation smgr state needs a reset, and the proposal does not state that rule anywhere a future author would see it. In [P1], removing an smgrclose() also skipped resetting smgr_cached_nblocks and set up a convention that smgrclose() may be left out in one special case. The same kind of convention appears here.

**Precedents.**
- [Remove duplicated smgrclose call](https://commitfest.postgresql.org/patch/5149/) (Returned with feedback, 2025-08-05); thread: [link](https://www.postgresql.org/message-id/flat/CABBtG=cDTCBDCBK7McSy6bJR3s5xUTOg0vSFfuW8oLdUYyCscA@mail.gmail.com)

Thread: [#1 (Bryan Green, 2026-10-26)](https://www.postgresql.org/message-id/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com)

**Question:** What invariant makes the size-only message safe for vm_extend(), and where is it documented or asserted so a later caller cannot use it after truncation?

Counter-check: held. The proposal does not state the rule that makes the size-only message safe, and message #1 itself asks whether the full invalidation is needed. [P1] concerns duplicate smgrclose() calls after unlink and should not be used as support.

### 2. New invalidation message type is unsafe to back-patch across minor versions

fixable · level 2, rubric item 6 · backpatchability, WAL / replication / standby, ABI / extension API · `replication-ondisk-1`

A user runs a primary on the patched minor release and a hot standby on the previous one. The primary writes a commit or XLOG_INVALIDATIONS record that carries the new message ID. The older standby has no handler for that ID, so replay or logical decoding errors out, and the user finds out when the standby stops applying WAL.

The patch touches standbydesc.c, which describes invalidation messages carried in WAL, so the new ID may be written to WAL records. Message #1 notes the problem exists in every branch from 14 to master. The proposal does not say whether it targets those branches or master only. It also does not say whether the new ID always stays out of WAL; vm_extend() sends its invalidation directly, but the material does not confirm the new ID never enters the transactional queue.

Thread: [#1 (Bryan Green, 2026-10-26)](https://www.postgresql.org/message-id/62fd95fd-d81d-4566-998b-dc6e9422e9f1%40gmail.com)

**Question:** Can the new message ID ever be written to WAL, and if so, is the patch meant for master only or for back-branches too?

Counter-check: held. The smgr invalidation is sent outside the transactional queue today, which suggests the new ID stays out of WAL. But the patch touches standbydesc.c and message #1 names branches 14 to master without stating a target, so the question stands.

### 3. Point fix for one source of a general reopen cost

fixable · level 1b, rubric item 5 · design, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-3`

After this lands, the same encrypted-filesystem user still sees the inserts stall when DDL, truncation or other relcache and smgr invalidations hit a large table. Each source then gets its own special message, and the number of invalidation variants grows release by release.

Message #2 says DDL invalidations cause the same load and asks whether Postgres should optimize for costly open() at all. That question has no answer in the thread. A committer would likely ask whether the cost belongs at the smgr or fd layer, such as keeping descriptors across an invalidation when segments still exist, so that every invalidation source gains instead of one.

Thread: [#2 (Jakub Wartak, 2026-10-27)](https://www.postgresql.org/message-id/CAKZiRmxmPL_7grAoEF6mxuyWXUi8BCnZZpaEaF05AX2NDfFP0g%40mail.gmail.com)

**Question:** Why is a vm_extend()-specific message better than making smgr invalidation keep open segments in general, and which other sources would still cause mass reopens?

Counter-check: held. Message #2 raises this question and the thread has no answer. It is weaker than stated, because vm_extend() fires routinely during VACUUM on large tables while DDL on them is rare, so a targeted fix may still be justified.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch touches standbydesc.c, which describes invalidation messages carried in WAL records. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a message type in sinval.h and changes declarations in smgr.h and inval.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #1 reports the problem on 17 and names branches 14 to master, but states no target branch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second, weaker smgr invalidation kind exists that later callers must choose correctly. |
| performance on the hot path | **yes** | The patch changes what every backend does on receiving the invalidation that vm_extend() sends during VACUUM and pruning. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #2 asks whether Postgres should optimize for filesystems with costly open() at all. |

## 5. Questions the author must answer before anyone reviews code

1. What invariant makes it safe for other backends to keep their segment files open after vm_extend(), and where does the patch state or assert it?
2. Can the new message ID ever be written to a WAL record, such as a commit or XLOG_INVALIDATIONS record?
3. Is the patch meant for master only, or for back-branches down to 14?
4. Why is a vm_extend()-specific message better than making the smgr invalidation keep existing segments open in general, and which other sources would still cause mass reopens?
5. Does a benchmark on ext4 or XFS show no regression from the new message?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the smgr and md layers and shared invalidation handling, including how invalidation messages reach WAL and standbys. No such committer has spoken in the two visible messages.
- **Liability:** The owner signs for the claim that no backend needs its files closed when the visibility map grows, on primaries and standbys, for as long as the message type exists. A back-patch adds minor-version compatibility of the new message ID.
- **Author follow-up:** The author tested an alternative design and asked the list for the missing reason in message #1; no later sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The measured gain is large but shows on filesystems with costly open(), large tables and many writers; other users only lose unneeded reopens. The patch is small, about 10 KiB over seven files, but it adds a permanent invalidation kind in smgr and sinval code that needs expert review.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` New sinval message type may not be safe to back-patch to 14. Refuted (not specific): Duplicate of replication-ondisk-1, which asks the same back-patch and WAL question and also notes that vm_extend() sends its invalidation directly.
- `maintenance-fit-2` Fix needs a new message type but the reporter runs 17. Refuted (not specific): Duplicate of replication-ondisk-1. The added point that a master-only fix does not help a user on 17 is true of any non-back-patched improvement and is not a design flaw.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed this idea, and P1's reason does not carry over because this patch still resets the cached fork sizes. Message #2 says commit b4e074172768 first sent this invalidation to fix a regression, but it reports no reason that the files must be closed.
- standards-compat: 1 objection(s). The proposal changes no SQL, GUC, catalog, pg_dump or psql surface. The only user-visible changes are pg_waldump text and the back-patch question above.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format. Visibility map extension during standby replay runs its own invalidation in the startup process, so the standby keeps today's behaviour.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes an internal shared invalidation message that only the server sends, adds no SQL surface, function, setting or view, and exposes nothing new to unprivileged users. It reduces file-open load rather than adding a denial-of-service path. The risk that a back-patched new message type in commit records reaches an older standby is a compatibility question for the replication and backpatch angle.
- maintenance-fit: 3 objection(s). Objections found; the change itself removes work for all users, so no hot-path objection is raised.
- already-exists: 0 objection(s). Postgres has no existing way to tell other backends a fork grew without making them close their files. The only cross-backend message for this is the full smgr invalidation that the patch replaces, and no extension or hook can change what vm_extend() sends. The new message is a narrower form of an existing mechanism, so it is not a second way to do the same job. Whether it is safe and wanted is for the design and desirability angles to judge.
- demand: 0 objection(s). The material shows real demand. Message #1 reports a production stall on a 4TB partition on 17, with numbers, and Jakub Wartak independently reproduced high system CPU on encrypted filesystems in message #2, naming three such Linux filesystems. The change also drops unneeded file reopens for every user, so the cost does not fall mainly on non-users. Jakub's broader question about other sinvals is a scope question, not missing demand.

### Precedents retrieved

- P1: [Remove duplicated smgrclose call](https://commitfest.postgresql.org/patch/5149/) (cf, Returned with feedback, 2025-08-05) - used in the review; reason summarised from its thread
- P2: [Use function smgrclose() to replace the loop](https://commitfest.postgresql.org/patch/5196/) (cf, Withdrawn, 2025-12-02) - judged not relevant; reason summarised from its thread
- P3: [Windows pg_basebackup unable to create >2GB pg_wal.tar tarballs ("could not close file: Invalid argument" when creating pg_wal.tar of size ~ 2^31 bytes)](https://commitfest.postgresql.org/patch/5403/) (cf, Withdrawn, 2024-11-25) - judged not relevant; reason summarised from its thread
- P4: [Use simplehash.h instead of dynahash in SMgr](https://commitfest.postgresql.org/patch/3092/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P5: [Revert "Use "transient" files for blind writes, take 2".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9bacf0e3734f0b5dc821abf7cb312377876e22d3) (revert, Reverted, 2012-10-17) - used in the review
- P6: [warn if GUC set to an invalid shared library](https://commitfest.postgresql.org/patch/3482/) (cf, Returned with feedback, 2024-12-11) - judged not relevant; reason summarised from its thread
- P7: [Windows filesystem support improvements](https://commitfest.postgresql.org/patch/3951/) (cf, Returned with feedback, 2023-03-03) - judged not relevant; reason summarised from its thread
- P8: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/2273/) (cf, Returned with feedback, 2019-11-29) - judged not relevant; reason summarised from its thread
- P10: [Write visibility map during CLUSTER/VACUUM FULL](https://commitfest.postgresql.org/patch/3206/) (cf, Returned with feedback, 2022-06-30) - judged not relevant
- P11: [Reopen logfile on SIGHUP](https://commitfest.postgresql.org/patch/1558/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P12: [Custom storage managers (SMGR), redux](https://commitfest.postgresql.org/patch/4428/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P13: [Restricting maximum keep segments by repslots](https://commitfest.postgresql.org/patch/1260/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P14: [makes pgbench functions available on server side - Common Library Extraction (patch 1 - refactoring only) & Server Extension (patch 2 adds extension)](https://commitfest.postgresql.org/patch/7225/) (cf, Withdrawn, 2026-10-20) - judged not relevant
- P15: [Don't keep closed WAL segment in page cache after replay](https://commitfest.postgresql.org/patch/5894/) (cf, Withdrawn, 2026-08-20) - judged not relevant
- P16: [Extended statistics in EXPLAIN](https://commitfest.postgresql.org/patch/3050/) (cf, Returned with feedback, 2022-08-01) - judged not relevant
- P17: [WIP: Track relation size in shared memory](https://commitfest.postgresql.org/patch/2933/) (cf, Returned with feedback, 2021-10-01) - used in the review
- P18: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P19: [OFFSET optimisation for IndexScan using visibility map](https://commitfest.postgresql.org/patch/1513/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P20: [invalid data in file backup_label problem on windows](https://commitfest.postgresql.org/patch/2938/) (cf, Withdrawn, 2021-04-06) - judged not relevant
- P21: [Not to invalidate CatalogSnapshot for local invalidation messages](https://commitfest.postgresql.org/patch/4739/) (cf, Rejected, 2024-04-08) - judged not relevant
- P22: [list of extended statistics on psql](https://commitfest.postgresql.org/patch/2692/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P23: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - judged not relevant
- P24: [Close backend file handles on exec](https://commitfest.postgresql.org/patch/886/) (cf, Returned with feedback, 2017-01-31) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-3); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 134 s; cost reported by the CLI: $1.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:03:52.532Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
