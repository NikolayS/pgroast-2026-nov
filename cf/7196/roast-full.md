# Design pre-review: Improve cube GiST page splits

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7196](https://commitfest.postgresql.org/patch/7196/) (Needs review); authors: Andrey Borodin (x4m)
- Thread: [Improve cube GiST page splits](https://www.postgresql.org/message-id/flat/3F6C9E73-9A6C-409D-9734-3D7C4E7FEA81%40yandex-team.ru), started 2026-08-09
- Messages read: 3 of 3
- Patch prose read: v2-0001-Add-regression-coverage-for-cube-GiST-page-splits.patch, v2-0002-Use-double-sorting-for-cube-GiST-page-splits.patch, 0001-Add-regression-coverage-for-cube-GiST-page-splits.patch, 0002-Use-double-sorting-for-cube-GiST-page-splits.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived the critic. The only one raised, maintenance-fit-1, was refuted because each GiST opclass already keeps its own picksplit. The patch fixes a measured defect in an existing contrib opclass, copies the box and point approach, and adds no user-visible surface. Confidence is medium because nobody has reviewed v2 and no numbers exist for v2.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the v2 handling of NaN, infinite and extreme coordinates could not be checked. Split quality of v2 and split CPU cost at many dimensions are also unverified.

## The proposal, restated

**Problem.** The cube GiST opclass uses Guttman's polynomial split, which can put almost all entries on one side of a split. With sorted input, insertion becomes quadratic and the index grows very large. This affects users of contrib/cube with GiST indexes.

**Proposed design.** The patch replaces the cube picksplit with the double-sorting split used by the built-in box and point GiST opclasses, generalized to any number of dimensions. Absent dimensions count as zero, the same minimum split ratio applies, and an even split is the fallback. v2 uses float8 ordering for bounds and non-throwing C arithmetic for split-quality estimates.

**User-visible surface.**
- Changed on-disk shape and size of existing and new cube GiST indexes; no new SQL objects, GUCs or C API.

**What the author claims.**
- For 4000 sorted 2-D cubes, insertion drops from about 111 s to about 0.02 s and the index from 33712 to 36 pages.
- The reviewer reports patched indexes are never larger than master at 20k rows across eight data shapes, with equal or fewer buffers per query.
- v2 fixes the NaN "division by zero" and the large-coordinate overflow errors found in review.
- v2 adds regression tests for NaN, infinite, extreme finite and underflow coordinates, for both insertion and create index.

**Thread state.** Rustam Allakov reviewed v1: index scans agreed with seq scans, but NaN and very large coordinates made inserts and create index fail with an error. Andrey posted v2 with fixes and tests on 2026-11-02; nobody has reviewed v2 yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-08-09; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | New splits change the shape and size of cube GiST indexes; the tuple and page format stay the same. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Message #2 showed v1 made CREATE INDEX fail on restore for NaN and large coordinates; v2 says it fixes this. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | v2-0002 adds an N-dimensional double-sorting split to contrib/cube/cube.c, separate from the box and point code. |
| performance on the hot path | **yes** | The patch replaces the picksplit that runs on every cube GiST page split, for cube users only. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does v2 keep the v1 page counts and buffers per query on the eight data shapes from message #2?
2. How does insertion time and CREATE INDEX time compare with master for random, unsorted input?
3. What is the CPU cost of one split at high dimensionality, up to the cube maximum of dimensions?
4. Is there any input that master indexes and v2 still rejects with an ERROR, beyond the cases covered by the new tests?
5. Can the commit message state that existing cube GiST indexes need no reindex and that old and new splits can coexist in one index?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows GiST picksplit and the float8 edge cases (NaN, infinity, overflow) in the geometric opclasses would have to own it. None has spoken in the visible thread.
- **Liability:** The owner signs for the cube split behaviour on all inputs, including NaN and extreme coordinates, and for any later report of worse index quality on some data shape. The exposure is limited to contrib/cube GiST users.
- **Author follow-up:** The author answered the review in message #3 with v2, fixes for both reported errors and new tests, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch touches one file in contrib/cube plus tests and adds no SQL objects, GUCs or C API, and message #1 shows a very large gain for sorted input. Review cost is medium because the float8 edge cases need care and v2 is unreviewed.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Second copy of the double-sorting split, already diverging from box. Refuted (no failure scenario): Each GiST opclass already keeps its own picksplit, and range types have a separate double-sorting split in rangetypes_gist.c, so the patch follows existing practice. Box and cube already split differently on master, and a missed split fix changes index quality only, not query results.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] is the same author's 2016 change to the cube penalty function, which is a different proposal; its recorded reason, IEEE 754 bit tricks inside the float penalty, does not apply to a picksplit change. [P17] is about NaN in geo_ops and records no reason. Message #1 links a 2016 discussion where the author first saw unbalanced cube trees, and the thread reports no rejection of a split change from it.
- standards-compat: 0 objection(s). The patch adds no SQL objects, keywords, GUCs, psql features or C API. It changes only how cube GiST pages split, so existing query results stay the same, and old and new indexes are both valid GiST trees that pg_upgrade keeps without a reindex. Message #2 showed that v1 broke pg_dump restore: CREATE INDEX failed on NaN and large coordinates that master accepts. Message #3 says v2 fixes this and adds tests for insertion and CREATE INDEX. Nobody has checked v2 yet. If a v2 review finds any input that master indexes and v2 rejects, the pg_dump round-trip objection applies again.
- replication-ondisk: 0 objection(s). The patch changes only how cube picksplit distributes entries. The index tuple format, WAL records, catalog, extension SQL objects and C API stay the same. GiST correctness does not depend on the split algorithm, so existing indexes stay valid after pg_upgrade without a reindex, and pages split by the old and new code can coexist in one index. Standbys replay the resulting pages and never run picksplit, so physical replication is not affected, and neither are mixed-version standbys. Message #2 found a dump-and-restore failure on NaN and extreme coordinates, and v2 says it fixes it with tests. That claim concerns the implementation, and nobody has reviewed v2 yet, so it could not be checked from this angle.
- security: 0 objection(s). The patch changes only the picksplit method of the cube GiST opclass. It adds no SQL objects, privileges, GUCs or monitoring output, so nobody gains new access or new information. A split handles at most one page of entries, so its sorting cost is bounded and does not create a new denial-of-service path. The ERRORs that blocked inserts and restores were reported in message #2 as robustness problems, and v2 says it fixes them. Whether that fix holds needs a check of the code, which this angle does not judge.
- maintenance-fit: 1 objection(s). Within the maintenance and fit angle, costs fall only on users of contrib/cube with GiST indexes, the design copies existing core practice, and no new GUC, catalog, API or dependency is added. No committer has engaged in the visible thread, which affects committability, not the merits.
- already-exists: 0 objection(s). No existing feature does this job. Postgres offers no way to swap the picksplit of the cube GiST opclass, and the box opclass covers only two dimensions. The patch reuses the double-sorting split from the box and point opclasses inside an existing contrib module, so it adds no second mechanism and needs no new hook. Precedent P3 changed the penalty function and was returned over IEEE 754 bit-twiddling, which does not apply here.
- demand: 0 objection(s). From the demand angle there is no objection to send. The patch fixes a measured defect in an existing contrib opclass and adds no SQL objects, GUCs or C API. Its cost falls only on contrib/cube GiST users, who also get the benefit. Message #1 shows sorted input making insertion quadratic and the index about 1000 times larger. In message #2 the reviewer found smaller or equal indexes and fewer buffers per query across eight data shapes, including random ones. No field reports are cited, but a defect fix in a shipped opclass does not need the demand evidence a new feature needs. Not checked here: CPU cost of the split at high dimensionality, and whether v2 keeps the v1 split quality.

### Precedents retrieved

- P1: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P2: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P3: [Cube extension GiST penalty function improvement](https://commitfest.postgresql.org/patch/782/) (cf, Returned with feedback, 2016-10-03) - used in the review; reason summarised from its thread
- P4: [Cube extension kNN support](https://commitfest.postgresql.org/patch/138/) (cf, Returned with feedback, 2015-07-07) - judged not relevant; reason summarised from its thread
- P5: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P6: [sort leaf pages by ctid for gist indexes built using sorted method](https://commitfest.postgresql.org/patch/3454/) (cf, Returned with feedback, 2022-03-22) - judged not relevant; reason summarised from its thread
- P7: [Sort leaf pages by ctid for gist indexes built using sorting method](https://commitfest.postgresql.org/patch/3455/) (cf, Withdrawn, 2021-12-16) - judged not relevant; reason summarised from its thread
- P8: [New gist vacuum](https://commitfest.postgresql.org/patch/371/) (cf, Returned with feedback, 2016-01-31) - judged not relevant
- P9: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P10: [Delete pages during GiST VACUUM](https://commitfest.postgresql.org/patch/1416/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P11: [Add sortsupport for range types and btree_gist](https://commitfest.postgresql.org/patch/3686/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P12: [Split xlog.c](https://commitfest.postgresql.org/patch/3173/) (cf, Returned with feedback, 2021-11-15) - judged not relevant; reason summarised from its thread
- P13: [\[PATCH\] reduce page overlap of GiST indexes built using sorted method](https://commitfest.postgresql.org/patch/3487/) (cf, Returned with feedback, 2022-01-09) - judged not relevant
- P14: [Parallel CREATE INDEX for GiST indexes](https://commitfest.postgresql.org/patch/5026/) (cf, Returned with feedback, 2025-02-06) - judged not relevant
- P15: [Revert "Add sortsupport for gist_btree opclasses, for faster index builds."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d92b1cdbab408d8f1299257125c9ae375f3ca644) (revert, Reverted, 2021-04-07) - judged not relevant
- P16: [Compress prune/freeze records with Delta Frame of Reference algorithm](https://commitfest.postgresql.org/patch/6574/) (cf, Returned with feedback, 2026-07-07) - judged not relevant
- P17: [Fix behavior of geo_ops when NaN is involved](https://commitfest.postgresql.org/patch/2710/) (cf, Returned with feedback, 2022-08-01) - used in the review
- P18: [Split StdRdOptions into HeapOptions and ToastOptions](https://commitfest.postgresql.org/patch/2370/) (cf, Returned with feedback, 2021-10-04) - judged not relevant
- P19: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P20: [Improved crc32c performance with AVX-512 algorithm.](https://commitfest.postgresql.org/patch/5100/) (cf, Returned with feedback, 2025-04-02) - judged not relevant
- P21: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P22: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P23: [Optimize numeric.c mul_var() using the Karatsuba algorithm](https://commitfest.postgresql.org/patch/4930/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P24: [Add a greedy join search algorithm to handle large join problems](https://commitfest.postgresql.org/patch/6486/) (cf, Withdrawn, 2026-02-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 100 s; cost reported by the CLI: $0.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 48 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:27:29.366Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
