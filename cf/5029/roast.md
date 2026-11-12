# Design pre-review: Historic snapshot doesn't track txns committed in BUILDING_SNAPSHOT state

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5029](https://commitfest.postgresql.org/patch/5029/); [thread](https://www.postgresql.org/message-id/flat/tencent_6AAF072A7623A11A85C0B5FD290232467808%40qq.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal: message #2 agrees this is a bug, and the fix adds no user-visible surface. The one surviving objection, maintenance-fit-1, is a fixable design point about a separate list of catalog-marking WAL records in the BUILDING_SNAPSHOT path. The visible material does not show v4-0001, so it is unknown whether that point is already answered. What remains is a correctness judgment in the snapshot builder that needs a committer who knows logical decoding.

Not checked: The code and v4-0001 were not read, so the correctness of tracking commits before FULL_SNAPSHOT is not judged. It was also not checked whether snapshot files in pg_logical/snapshots, written by unfixed binaries, keep the bad state after a minor upgrade.

## Prior attempts

None found before 2024-06-09.

## Objections a reviewer is likely to raise

### 1. Hand-kept list of catalog-marking WAL records in the BUILDING_SNAPSHOT path

A later patch adds or changes a WAL record that marks a transaction as catalog modifying and updates the FULL_SNAPSHOT decode path only. Slots created while such transactions commit get a snapshot that misses the change. Users see the same "could not map filenumber" error, and no test or assertion catches the gap.

v3-0001 adds a second, state-specific branch in decode.c that handles only XLOG_HEAP2_NEW_CID and XLOG_HEAP_INPLACE, with the comment "Currently only XLOG_HEAP2_NEW_CID means a catalog modifying". Message #8 suggests routing BUILDING_SNAPSHOT through the existing record handlers, so that future changes there apply here too. Message #9 asks why the patch calls ReorderBufferXidSetCatalogChanges and does not use SnapBuildProcessNewCid. The visible material does not show that v4-0001 removes the duplicated list or answers either point.

**Question:** Does v4-0001 reuse the existing record handlers so that a new catalog-marking record is covered without a separate BUILDING_SNAPSHOT edit, and if not, what catches the omission?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#8](https://www.postgresql.org/message-id/Zrmh7X8jYCbFYXjH%40ip-10-97-1-34.eu-west-3.compute.internal), [#9](https://www.postgresql.org/message-id/CAFPTHDZaKuS8j72C5s63Hisq6ZRCUu4%2BxTRSc31_vHgRhaHxzg%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/tencent_358CCA18E3191FF8D955865A17F6D49EA50A%40qq.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Does v4-0001 reuse the existing record handlers during BUILDING_SNAPSHOT, and if not, what catches a new catalog-marking record that is added only to the FULL_SNAPSHOT path?
2. Why does the patch call ReorderBufferXidSetCatalogChanges for XLOG_HEAP2_NEW_CID and not SnapBuildProcessNewCid, as message #9 asks?
3. Which of the three parallel threads holds the current patch, and can the review continue in one thread?
4. Which branches should receive the fix, and does the patch apply to them without changes that affect extensions?
5. Why did v1 change snapbuild.c while v3 changes only decode.c?

## Touch points

- **WAL / replication / standby:** The patch changes how decode.c handles commit, XLOG_HEAP2_NEW_CID and XLOG_HEAP_INPLACE records before FULL_SNAPSHOT.
- **backpatchability:** This is a bug fix in logical decoding, and the thread does not say which branches receive it.
- **maintenance burden ("who maintains this in 10 years"):** v3-0001 adds a BUILDING_SNAPSHOT branch with its own list of catalog-marking records that later patches must keep in sync.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the snapshot builder and the reorder buffer in logical decoding. Masahiko Sawada is listed as a reviewer on the CommitFest entry but has not posted in the visible thread.
- Priority: needs champion; audience a sizeable minority; review cost medium. The bug is agreed in message #2 and the patch is small, but the entry has been in 12 CommitFests with no committer posting on the design. The patch is small, but snapshot builder state changes need careful review, and the work is split across three threads.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
