# Design pre-review: Honor fillfactor in sorted GiST builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7198](https://commitfest.postgresql.org/patch/7198/); [thread](https://www.postgresql.org/message-id/flat/CAHqSB9jqtS94e9%3D0vxqQX5dxQA89N95UKyz-%3DA7Y%2B_YJt%2BVW5A%40mail.gmail.com), 17 of 17 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The goal stands: the patch makes a documented GiST reloption work again for sorted builds, and no surviving objection disputes it. Two fixable measurement gaps remain. maintenance-fit-1 asks for an unpatched baseline at the default fillfactor, and maintenance-fit-2 asks for the 2022 query benchmarks rerun on patched indexes. Confidence is low because the v1 patch prose is not in the material and both objections rest on missing numbers.

Not checked: The v1 patch, its touched files and any documentation hunk are not in the material. Whether the soft limit preserves the 2022 partitioning at the default fillfactor can only be judged from the code and new measurements.

## Prior attempts

None found before 2021-12-24.

## Objections a reviewer is likely to raise

### 1. Default fillfactor would change every sorted-built GiST index

A PostGIS user who never set fillfactor upgrades and runs reindex. Their index is now built under the default GiST fillfactor of 90 as a soft limit, where today the setting is ignored. Index size and page overlap both change, and the user first sees it as a larger index or a different query plan cost.

The patch helps users who set a non-default fillfactor, but the default applies to every GiST index with sortsupport, so most of the effect lands on people who never asked for it. Message #17 reports 220 pages at fillfactor 90 and 442 at 40, but gives no baseline for unpatched master. Without that baseline a reviewer cannot tell whether default builds stay the same size. The thread also does not say whether the change is for master only.

**Question:** For the same data and the default fillfactor, how do index size and page count compare between unpatched master and the patch?

<sub>fixable, level 1b, performance on the hot path; messages [#14](https://www.postgresql.org/message-id/fbbfe5dc-3dfa-d54a-3a94-e2bee37b85d8%40gmail.com), [#17](https://www.postgresql.org/message-id/0C45B219-9313-49DE-8E85-01C6CA9B5421%40yandex-team.ru); `maintenance-fit-1`</sub>

### 2. No query or overlap numbers for the change to the 2022 partitioning

The soft fillfactor limit cuts into the picksplit-based partitioning that f1ea98a79 added to reduce page overlap. If the new page boundaries raise overlap, spatial joins on rebuilt indexes slow down. The 2022 thread reported a slowdown of about 2 times in a self-join for the old sorted build, and nobody would notice a regression like that until queries slow down in production.

Messages #1, #4 and #12 justified the current algorithm with query timings and overlap statistics. The v1 patch in message #17 changes the same partitioning step but reports only page counts. Andrey Borodin noted in message #2 that picksplit makes an exact fillfactor impossible, so the soft limit interacts with picksplit in ways that page counts alone do not show. Few people know this code, and the visible thread shows no reviewer for v1.

**Question:** With the 2022 benchmarks rerun, do the self-join and tiling query times on patched default-fillfactor indexes match those on current master?

<sub>fixable, level 2, performance on the hot path; messages [#1](https://www.postgresql.org/message-id/CAHqSB9jqtS94e9%3D0vxqQX5dxQA89N95UKyz-%3DA7Y%2B_YJt%2BVW5A%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/23731640439330%40vla1-4ea76ba32639.qloud-c.yandex.net), [#4](https://www.postgresql.org/message-id/3aa6ba30-e9d8-10ef-753f-8deea5f196d0%40kontur.io), [#12](https://www.postgresql.org/message-id/47baa55e-f44f-40c2-f811-7d7211015418%40kontur.io), [#17](https://www.postgresql.org/message-id/0C45B219-9313-49DE-8E85-01C6CA9B5421%40yandex-team.ru); `maintenance-fit-2`</sub>

## Questions for the author

1. For the same data and the default fillfactor, how do index size and page count compare between unpatched master and the patch?
2. With the 2022 benchmarks rerun, do the self-join query times on patched default-fillfactor indexes match those on current master?
3. How does create index time for a sorted GiST build compare before and after the patch?
4. Is the change for master only, or is a backpatch proposed?
5. What will the docs and release notes tell users whose GiST fillfactor settings have had no effect since the 2022 change?

## Touch points

- **on-disk format / pg_upgrade:** see objection maintenance-fit-1: Default fillfactor would change every sorted-built GiST index
- **backpatchability:** The patch restores behaviour lost in 2022, and the thread does not say whether it is for master only.
- **maintenance burden ("who maintains this in 10 years"):** The sorted build in gistbuild.c gains a soft fillfactor limit that later changes to the partitioning step must keep working.
- **performance on the hot path:** The patch changes the picksplit-based partitioning that decides page overlap, and so scan time, for every sorted-built GiST index.

## Who would commit it, and when

- Owner: silent. Someone who knows the GiST build code, in particular the sorted build and its picksplit-based partitioning in gistbuild.c.
- Priority: proceed; audience niche; review cost medium. Only users who set a non-default fillfactor on sortsupport GiST indexes gain, and the change adds no new user surface. Review cost is a guess because the v1 patch was not read; the open points are measurements the author can supply.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
