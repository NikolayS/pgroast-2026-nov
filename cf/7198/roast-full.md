# Design pre-review: Honor fillfactor in sorted GiST builds

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7198](https://commitfest.postgresql.org/patch/7198/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://www.postgresql.org/message-id/flat/CAHqSB9jqtS94e9%3D0vxqQX5dxQA89N95UKyz-%3DA7Y%2B_YJt%2BVW5A%40mail.gmail.com), started 2021-12-24
- Messages read: 17 of 17
- Patch prose read: 02_reduce_page_overlap_of_gist_indexes_built_using_sorted_method.patch, reduce_page_overlap_of_gist_indexes_built_using_sorted_method.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The goal stands: the patch makes a documented GiST reloption work again for sorted builds, and no surviving objection disputes it. Two fixable measurement gaps remain. maintenance-fit-1 asks for an unpatched baseline at the default fillfactor, and maintenance-fit-2 asks for the 2022 query benchmarks rerun on patched indexes. Confidence is low because the v1 patch prose is not in the material and both objections rest on missing numbers.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v1 patch, its touched files and any documentation hunk are not in the material. Whether the soft limit preserves the 2022 partitioning at the default fillfactor can only be judged from the code and new measurements.

## The proposal, restated

**Problem.** Since commit f1ea98a79 (2022), sorted GiST builds ignore the fillfactor reloption, so the index size does not change with fillfactor. Alexander Lakhin found that a regression test meant to rebuild with a different fillfactor does nothing, and that docs and release notes never mentioned the change. This affects users who set fillfactor on GiST indexes whose opclass has sortsupport.

**Proposed design.** The patch treats fillfactor as a soft limit when the sorted build recursively partitions the buffered tuples into pages. A page may go over the limit when the opclass split would otherwise create a page with one tuple. Ordinary GiST page splits after the build do not change.

**User-visible surface.**
- The fillfactor reloption on GiST indexes now affects the size of indexes built with the sorted method.
- Index size after create index or reindex changes for GiST opclasses with sortsupport and a non-default fillfactor.

**What the author claims.**
- Supporting fillfactor is better than documenting sorted builds as an exception, more so as more opclasses provide sortsupport.
- With 20,000 points, fillfactor 90 gave 220 pages and fillfactor 40 gave 442 pages.
- A test covers large INCLUDE tuples at fillfactor 10.
- Ordinary GiST page splits are unchanged.

**Thread state.** Andrey Borodin posted v1 in August 2026 and asked whether it should go to the CommitFest, with no reply visible. The same day's Coverity issue in gist_indexsortbuild() and its Assert-only patch is a separate item.

## 2. Prior attempts

No precedent found (corpus searched for items before 2021-12-24; 22 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Default fillfactor would change every sorted-built GiST index

fixable · level 1b, rubric item 4 · performance on the hot path, on-disk format / pg_upgrade · `maintenance-fit-1`

A PostGIS user who never set fillfactor upgrades and runs reindex. Their index is now built under the default GiST fillfactor of 90 as a soft limit, where today the setting is ignored. Index size and page overlap both change, and the user first sees it as a larger index or a different query plan cost.

The patch helps users who set a non-default fillfactor, but the default applies to every GiST index with sortsupport, so most of the effect lands on people who never asked for it. Message #17 reports 220 pages at fillfactor 90 and 442 at 40, but gives no baseline for unpatched master. Without that baseline a reviewer cannot tell whether default builds stay the same size. The thread also does not say whether the change is for master only.

Thread: [#14 (Alexander Lakhin, 2023-04-05)](https://www.postgresql.org/message-id/fbbfe5dc-3dfa-d54a-3a94-e2bee37b85d8%40gmail.com), [#17 (Andrey Borodin, 2026-08-10)](https://www.postgresql.org/message-id/0C45B219-9313-49DE-8E85-01C6CA9B5421%40yandex-team.ru)

**Question:** For the same data and the default fillfactor, how do index size and page count compare between unpatched master and the patch?

Counter-check: held. Message #14 shows the current sorted build gives the same size at fillfactor 40 and 100, so the default of 90 has no effect today. Message #17 gives no unpatched baseline, so the material cannot show that default builds keep their size.

### 2. No query or overlap numbers for the change to the 2022 partitioning

fixable · level 2, rubric item 8 · performance on the hot path, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

The soft fillfactor limit cuts into the picksplit-based partitioning that f1ea98a79 added to reduce page overlap. If the new page boundaries raise overlap, spatial joins on rebuilt indexes slow down. The 2022 thread reported a slowdown of about 2 times in a self-join for the old sorted build, and nobody would notice a regression like that until queries slow down in production.

Messages #1, #4 and #12 justified the current algorithm with query timings and overlap statistics. The v1 patch in message #17 changes the same partitioning step but reports only page counts. Andrey Borodin noted in message #2 that picksplit makes an exact fillfactor impossible, so the soft limit interacts with picksplit in ways that page counts alone do not show. Few people know this code, and the visible thread shows no reviewer for v1.

Thread: [#1 (Aliaksandr Kalenik, 2021-12-24)](https://www.postgresql.org/message-id/CAHqSB9jqtS94e9%3D0vxqQX5dxQA89N95UKyz-%3DA7Y%2B_YJt%2BVW5A%40mail.gmail.com), [#2 (Andrey Borodin, 2021-12-25)](https://www.postgresql.org/message-id/23731640439330%40vla1-4ea76ba32639.qloud-c.yandex.net), [#4 ("sergei sh(dot)", 2022-01-09)](https://www.postgresql.org/message-id/3aa6ba30-e9d8-10ef-753f-8deea5f196d0%40kontur.io), [#12 ("sergei sh(dot)", 2022-01-26)](https://www.postgresql.org/message-id/47baa55e-f44f-40c2-f811-7d7211015418%40kontur.io), [#17 (Andrey Borodin, 2026-08-10)](https://www.postgresql.org/message-id/0C45B219-9313-49DE-8E85-01C6CA9B5421%40yandex-team.ru)

**Question:** With the 2022 benchmarks rerun, do the self-join and tiling query times on patched default-fillfactor indexes match those on current master?

Counter-check: held. The patch changes the partitioning step that f1ea98a79 justified with query timings in messages #1, #4 and #12. Message #17 reports only page counts, so it does not show that overlap and query time are preserved.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | see objection maintenance-fit-1: Default fillfactor would change every sorted-built GiST index |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The patch restores behaviour lost in 2022, and the thread does not say whether it is for master only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The sorted build in gistbuild.c gains a soft fillfactor limit that later changes to the partitioning step must keep working. |
| performance on the hot path | **yes** | The patch changes the picksplit-based partitioning that decides page overlap, and so scan time, for every sorted-built GiST index. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. For the same data and the default fillfactor, how do index size and page count compare between unpatched master and the patch?
2. With the 2022 benchmarks rerun, do the self-join query times on patched default-fillfactor indexes match those on current master?
3. How does create index time for a sorted GiST build compare before and after the patch?
4. Is the change for master only, or is a backpatch proposed?
5. What will the docs and release notes tell users whose GiST fillfactor settings have had no effect since the 2022 change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the GiST build code, in particular the sorted build and its picksplit-based partitioning in gistbuild.c.
- **Liability:** The owner signs for index size and scan performance of every sorted-built GiST index after reindex, including PostGIS indexes, for as long as the sorted build exists.
- **Author follow-up:** The author moved the entry to the next CommitFest in October 2026 and took part in the 2022 work on the same code.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Only users who set a non-default fillfactor on sortsupport GiST indexes gain, and the change adds no new user surface. Review cost is a guess because the v1 patch was not read; the open points are measurements the author can supply.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` No measurement shows honoring fillfactor helps sorted GiST indexes. Refuted (precedent mismatch): [P6] and [P8] rejected a new GIN reloption. Here fillfactor is an existing, documented GiST option that sorted builds honored in Postgres 14 (message #14). The remaining concerns about default-fillfactor size and missing query numbers duplicate maintenance-fit-1 and maintenance-fit-2.
- `standards-compat-1` Default fillfactor 90 silently changes every sorted GiST rebuild. Refuted (not specific): Duplicate of maintenance-fit-1 for the default-fillfactor size change, and of maintenance-fit-2 for the missing query and overlap numbers. [P8] concerned a new GIN option and should not be used as support.
- `standards-compat-2` Dormant fillfactor settings in dumps start to apply after restore. Refuted (no failure scenario): A user who set fillfactor = 10 gets the documented behaviour that Postgres 14 and the buffering build already give, which is not a failure. The CommitFest entry targets PG20 and no message proposes a backpatch, so the minor-release case is speculative.
- `already-exists-1` buffering = on already gives a GiST build that honors fillfactor. Refuted (no failure scenario): The scenario names a maintenance cost but no concrete failure. Item 12 points the other way here: the buffering build and Postgres 14 sorted builds honor fillfactor, so the patch removes a departure from existing practice and adds no new user surface.
- `demand-1` No user has asked for fillfactor in sorted GiST builds. Refuted (precedent mismatch): [P6] and [P8] asked for benchmarks before adding a new option, but this patch makes an existing documented option work again after f1ea98a79 silently disabled it. Message #15's "lack of field complaints" refers to a null dereference and says nothing about fillfactor demand.
- `demand-2` Default fillfactor may grow every sorted-built GiST index. Refuted (not specific): Duplicate of maintenance-fit-1, which states the same default-fillfactor concern and asks for the unpatched baseline.

### Review angles

- prior-attempts: 1 objection(s). No precedent proposes honoring fillfactor in sorted GiST builds. The GIN fillfactor rejections apply only in part, because the option already exists for GiST.
- standards-compat: 2 objection(s). Two objections, both about how existing settings behave after the change. The basic goal, making the reloption behave as documented, fits existing practice for other index types.
- replication-ondisk: 0 objection(s). The proposal changes only how a sorted GiST build fills pages, and the page layout stays the same, so existing indexes stay valid and pg_upgrade copies them unchanged. Standbys replay the built pages as before, and the visible material shows no catalog, WAL record, protocol or ABI change. The default fillfactor of 90 may grow sorted-built indexes and their build WAL after reindex; that is a size and performance question for another angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. Only the index owner can set fillfactor with alter index or create index, and the larger index at low fillfactor is the same kind of growth that buffering GiST builds and btree fillfactor already allow, so no new privilege, information leak or denial-of-service path appears.
- maintenance-fit: 2 objection(s). The idea is small and makes an existing reloption work as documented, so no fatal objection from this angle. The objections above ask for measurements and do not dispute the goal. The GIN fillfactor precedents [P6] and [P8] are not cited: they added a new option, while this patch makes an existing one work.
- already-exists: 1 objection(s). One objection from this angle. No known extension provides this, and the existing workaround is slower, so the gap is convenience and build speed, not capability.
- demand: 2 objection(s). Two demand objections stand. Fixing the documentation gap, for example by stating that sorted builds ignore fillfactor, remains a cheap alternative until demand is shown.

### Precedents retrieved

- P1: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P2: [Speeding up GIST index builds using popcount](https://commitfest.postgresql.org/patch/3023/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P3: [Better tracking of free space during SP-GiST index build](https://commitfest.postgresql.org/patch/739/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P4: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - used in the review; reason summarised from its thread
- P5: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P6: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - used in the review; reason summarised from its thread
- P7: [\[doc\] vacuum full, fillfactor, and "extra space"](https://commitfest.postgresql.org/patch/2412/) (cf, Returned with feedback, 2020-08-02) - judged not relevant; reason summarised from its thread
- P8: [Fillfactor for GIN indexes](https://commitfest.postgresql.org/patch/13/) (cf, Rejected, 2015-01-16) - used in the review; reason summarised from its thread
- P9: [Pageinspect - add functions on GIN and GiST indexes from gevel](https://commitfest.postgresql.org/patch/1207/) (cf, Returned with feedback, 2017-11-27) - judged not relevant; reason summarised from its thread
- P10: [Delete pages during GiST VACUUM](https://commitfest.postgresql.org/patch/1416/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P11: [Add IGNORE NULLS and FROM LAST options](https://commitfest.postgresql.org/patch/1724/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P12: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P13: [Revert 95d737ff to add 'ignore_nulls'](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8a026e4f15e2063cd7fe6cb9ffe44f2a81398bd) (revert, Reverted, 2014-09-29) - judged not relevant
- P14: [Ignore attributes of tables part of extensions in pg_dump](https://commitfest.postgresql.org/patch/164/) (cf, Rejected, 2015-02-23) - judged not relevant
- P15: [support for box type in SP-GiST index](https://commitfest.postgresql.org/patch/419/) (cf, Returned with feedback, 2016-01-28) - judged not relevant
- P16: [Revert "Include information on buffer usage during planning phase, in EXPLAIN output."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=19db23bcbda99e93321cb0636677ec9c6e121a2a) (revert, Reverted, 2020-04-03) - judged not relevant
- P17: [Revert "Move page initialization from RelationAddExtraBlocks() to use."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=684200543b4cbfe1ac002c9962e90683d4ea4691) (revert, Reverted, 2019-01-29) - judged not relevant
- P18: [New gist vacuum](https://commitfest.postgresql.org/patch/371/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P19: [Predicate locking in SP-Gist index](https://commitfest.postgresql.org/patch/1215/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P20: [SortSupport implementation on inet/cdir](https://commitfest.postgresql.org/patch/1990/) (cf, Withdrawn, 2019-02-10) - judged not relevant
- P21: [Move all am-related reloption code into src/backend/access/\[am-name\] and get rid of relopt_kind for custom AM](https://commitfest.postgresql.org/patch/992/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P22: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (maintenance-fit-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 158 s; cost reported by the CLI: $2.25
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 3 of 22 kept
- Tone check: 84 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:33.468Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
