# Design pre-review: BUG #19603: NaN coordinates in cube type break comparison and GiST scans

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7107](https://commitfest.postgresql.org/patch/7107/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [BUG #19603: Vuln47: distance_taxicab and distance_chebyshev silently return 0 instead of NaN when a cube coordin](https://www.postgresql.org/message-id/flat/19603-7b1f783d5bbfe791%40postgresql.org), started 2026-08-03
- Messages read: 9 of 9
- Patch prose read: 0001-cube-propagate-NaN-in-taxicab-Chebyshev-and-Euclidea.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the NaN semantics, which follow float8 practice. The two surviving objections, standards-compat-1 and replication-ondisk-1, are fixable rollout points. The patch changes the comparisons behind the cube btree operator class, the thread covers REINDEX only for GiST, and the entry targets stable. A committer has to decide between back branches and master only.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only the v1 commit message and 3 of 9 messages were visible, so the scope of v4, the GiST -Infinity lower bound logic and the cost of the added NaN checks could not be judged.

## The proposal, restated

**Problem.** A NaN coordinate in a cube value makes all float comparisons false, so distance_taxicab, distance_chebyshev and <-> return 0 or a finite value instead of NaN. The reviewer found that equality, ordering, cube_is_point, cube_size and GiST KNN scans are wrong in the same way. This affects users who store or query cube values with NaN coordinates.

**Proposed design.** distance_1D() returns NaN when any endpoint is NaN, and Chebyshev uses float8_max so NaN wins the running maximum. Comparisons and bound selection use the float8 helpers, so NaN sorts the way it does for float8. For GiST internal keys, a NaN distance becomes -Infinity so subtrees that hold finite values are still searched.

**User-visible surface.**
- distance_taxicab, distance_chebyshev and <-> return NaN when a cube has a NaN coordinate.
- Cube equality and ordering operators (=, <, >) change results for cubes with NaN coordinates.
- cube_is_point and cube_size change results for NaN coordinates written as point or as box.
- Existing GiST indexes on cube columns that contain NaN need REINDEX to give correct results.

**What the author claims.**
- NaN should propagate through distances the way IEEE 754 arithmetic does.
- NaN is not a valid lower bound for an internal GiST key that also covers finite values, so -Infinity is used.
- Replacing the library and restarting does not repair existing indexes; REINDEX does.
- The reindex need should go into the release notes.

**Thread state.** Aleksander Alekseev reviewed v1 to v3 and found more NaN bugs, a missing GiST test, removed tests and a split patch. Ayush posted v4 as one patch that restores the old tests; no one has replied to v4.

## 2. Prior attempts

### Fix behavior of geo_ops when NaN is involved (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2710/](https://commitfest.postgresql.org/patch/2710/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAGf+fX70rWFOk5cd00uMfa__0yP+vtQg5ck7c2Onb-Yczp0URA@mail.gmail.com)
- **Date:** decided 2022-08-01; started 2020-06-24
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane committed parts of the work in November 2020. The rest was rebased, and a reviewer was asked for comments in March 2021, but no reviewer answered that request or the expanded patch set posted in December 2021. Jacob Champion returned the entry in August 2022 for lack of reviewer interest, and said this was not a rejection.
- **Reason (quote):** "Rather than code changes, what this patch needs is more community interest." ([message](https://www.postgresql.org/message-id/f84318e2-2918-8c21-436f-a79823c9e097%40timescale.com))
- **What is different this time:** P1 made NaN behave consistently in core geo_ops. This proposal does the same for contrib/cube, starts from a bug report, and has an engaged reviewer.


## 3. Objections a reviewer is likely to raise

### 1. Back-patching changes cube btree and GiST ordering in minor releases

fixable · level 2, rubric item 6 · backpatchability, planner behaviour change, on-disk format / pg_upgrade · `standards-compat-1`

A user has a btree index or unique constraint on a cube column that holds NaN coordinates. After a minor upgrade, the new comparison function no longer matches the order the existing index was built in. Index scans then miss rows, or a unique index accepts values it used to reject, and the user sees no error.

The CommitFest entry targets "stable", but the patch changes `=`, `<` and `>` for cube, and these operators back the cube btree operator class. Message #5 covers REINDEX only for GiST, and no message in the thread mentions btree indexes. Before the patch '(1,1)' = '(nan,nan)' returned true (message #3), so a unique index built on the old behaviour can hold values that are inconsistent under the new behaviour. A committer would likely ask whether a silent change in index semantics belongs in a minor release, even as a bug fix.

Thread: [#3 (Aleksander Alekseev, 2026-11-04)](https://www.postgresql.org/message-id/CAJ7c6TM7CNrQShuPapof77A6SoP-qqjpQX2q1C3B0HT4749T6A%40mail.gmail.com), [#5 (Ayush Tiwari, 2026-11-04)](https://www.postgresql.org/message-id/CAJTYsWWQhEA_FS8%3DeKbZ%2BcmuAHhHXBv%3DPaGQjExZu0VKQSm6yw%40mail.gmail.com)

**Question:** Which index types on cube columns become invalid after this change, and why is a minor release the right place for that change rather than master only?

Counter-check: held. The CommitFest entry targets stable, and the patch changes the comparisons behind the cube btree operator class. The thread discusses REINDEX only around the GiST fix and does not weigh back branches against master only.

### 2. Btree indexes and unique constraints on cube need reindex too

fixable · level 2, rubric item 7 · on-disk format / pg_upgrade, backpatchability · `replication-ondisk-1`

A user has a btree index or a unique constraint on a cube column holding NaN coordinates, built with the old cube_cmp order. After the update, the new comparison order no longer matches the order of the index pages. Lookups miss rows and the unique check can admit duplicates, and nothing reports it unless the user runs amcheck.

Message #5 changes comparisons to the float8 helpers, so NaN now sorts above all finite values. Message #3 shows that the old code treated '(1,1)' and '(nan,nan)' as equal. Each btree index on cube built under the old order is therefore out of order for NaN keys. Message #5 asks for a release note about REINDEX only for GiST, and the brief notes that btree is not addressed.

Thread: [#3 (Aleksander Alekseev, 2026-11-04)](https://www.postgresql.org/message-id/CAJ7c6TM7CNrQShuPapof77A6SoP-qqjpQX2q1C3B0HT4749T6A%40mail.gmail.com), [#5 (Ayush Tiwari, 2026-11-04)](https://www.postgresql.org/message-id/CAJTYsWWQhEA_FS8%3DeKbZ%2BcmuAHhHXBv%3DPaGQjExZu0VKQSm6yw%40mail.gmail.com)

**Question:** Which index types and constraints on cube columns must be rebuilt after the fix, and how can a user find the affected ones before updating?

Counter-check: held. Cube has a btree operator class built on the comparison the patch changes, and no message mentions btree indexes or unique constraints. This holds for a master-only fix too, because pg_upgrade keeps index files.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Existing GiST and btree indexes on cube columns with NaN no longer match the new comparison and distance rules; pg_upgrade keeps index files. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | see objection standards-compat-1: Back-patching changes cube btree and GiST ordering in minor releases |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and the fix changes cube =, < and > and GiST distance results that existing indexes depend on. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | Cube comparisons and GiST distance calls gain NaN handling through the float8 helpers; no measurement is shown. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which index types and constraints on cube columns (btree, unique, GiST) must be rebuilt after the fix?
2. Is the comparison and ordering change meant for back branches, and if so, why is a REINDEX requirement in a minor release better than a master-only fix?
3. Could the distance fix go to back branches while the comparison and ordering change goes to master only?
4. How can a user find the cube indexes that hold NaN coordinates before updating?
5. Was rejecting NaN in cube input considered as an alternative, and why was it not chosen?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows GiST opclass support functions and back-patch policy for changes to index ordering. The visible thread shows a reviewer but no committer.
- **Liability:** The committer signs for the new NaN ordering in the cube btree and GiST opclasses, the REINDEX release note, and any later reports of wrong index scans, in every branch that gets the fix.
- **Author follow-up:** The author posted v2 to v4 in response to review, the latest within days of the reviewer's comments.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Only users with NaN coordinates in cube values benefit, but the current results are wrong and the change stays inside contrib/cube with existing float8 helpers. Review cost is medium because the GiST key logic and the back-branch decision need care, and v4 was not read.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Comparison change targets stable branches but invalidates existing indexes. Refuted (not specific): Duplicate of standards-compat-1. [P11] must not be used as support: that objection was to a GUC that switched comparison rules at run time, which this patch does not add.
- `standards-compat-2` No way to find which indexes need REINDEX. Refuted (no failure scenario): Rebuilding every cube index is a complete remedy, so an administrator is not left without a safe action. The wording of the release note is documentation detail, and replication-ondisk-1 already asks how users find affected indexes.
- `replication-ondisk-2` A stable-branch fix changes persisted index order in a minor release. Refuted (not specific): Duplicate of standards-compat-1. The standby case adds no new failure, since the indexes disagree with the new code on every node until REINDEX, and [P11] concerned a run-time GUC.
- `maintenance-fit-1` Back-patching changes cube btree and GiST ordering inside a minor release. Refuted (not specific): Duplicate of standards-compat-1, which makes the same back-branch and btree points without relying on [P11]. [P11] concerned a GUC that changed comparison rules, so it does not support this objection.
- `demand-1` Back-patching a NaN ordering change has no field demand behind it. Refuted (not specific): This is a fix for wrong results, and message #4 shows GiST scans over NaN cubes are already wrong, so the cost falls on the sites that gain. The back-branch question is a duplicate of standards-compat-1.

### Review angles

- prior-attempts: 1 objection(s). [P1] stalled for lack of review, and that reason does not apply here: a bug report and an active reviewer exist. [P15] records no reason, so it supports no objection.
- standards-compat: 2 objection(s). Both objections concern rollout and compatibility. Making NaN propagate and sort as it does for float8 is consistent with existing practice, and I raise no objection to the semantics themselves.
- replication-ondisk: 2 objection(s). The proposal has no WAL, catalog, wire protocol or ABI change. The objections above concern index contents that already exist on disk.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes the results of cube distance, comparison and GiST functions for NaN coordinates, and it adds no new object, privilege, function or setting. The "Vuln" label and CWE tag in message #1 describe wrong results, not any change in who can do or see what. The REINDEX need for btree and GiST indexes affects correctness and back-branch policy, which other angles cover. I could not check without the code whether any cube function is marked leakproof; if one is, its error behaviour on NaN should not change.
- maintenance-fit: 1 objection(s). Beyond the back-branch question, the maintenance cost looks small. The change stays inside contrib/cube and uses existing float8 helpers. The extra NaN checks on comparisons and GiST distance calls should cost little, but no measurement was shown, and the v4 code is not visible.
- already-exists: 0 objection(s). This is a correctness fix inside contrib/cube, and no existing feature, setting or extension makes cube distances, comparisons or GiST scans handle NaN correctly. The patch reuses the float8 comparison helpers, which follows existing practice and does not add a second mechanism.
- demand: 1 objection(s). A wrong-results fix in master needs no demand case, so the one objection covers only the back-patch to stable.

### Precedents retrieved

- P1: [Fix behavior of geo_ops when NaN is involved](https://commitfest.postgresql.org/patch/2710/) (cf, Returned with feedback, 2022-08-01) - used in the review; reason summarised from its thread
- P2: [Revert "Test conversion of NaN between float4 and float8."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5c4c771dafefa7d0ee80a6bd6955b7a3940452a1) (revert, Reverted, 2018-05-05) - judged not relevant
- P3: [Cube extension kNN support](https://commitfest.postgresql.org/patch/138/) (cf, Returned with feedback, 2015-07-07) - judged not relevant; reason summarised from its thread
- P4: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P5: [Cube extension GiST penalty function improvement](https://commitfest.postgresql.org/patch/782/) (cf, Returned with feedback, 2016-10-03) - judged not relevant; reason summarised from its thread
- P6: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant; reason summarised from its thread
- P7: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P8: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P9: [Optimize SnapBuild by maintaining committed.xip in sorted order](https://commitfest.postgresql.org/patch/6187/) (cf, Withdrawn, 2026-04-29) - judged not relevant; reason summarised from its thread
- P10: [KNN-spgist](https://commitfest.postgresql.org/patch/976/) (cf, Returned with feedback, 2017-03-24) - judged not relevant; reason summarised from its thread
- P11: [Floating point comparison inconsistencies of the geometric types](https://commitfest.postgresql.org/patch/679/) (cf, Returned with feedback, 2017-03-16) - used in the review; reason summarised from its thread
- P12: [ANALYZE: hash-accelerate MCV tracking for equality-only types](https://commitfest.postgresql.org/patch/6422/) (cf, Withdrawn, 2026-05-03) - judged not relevant
- P13: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P14: [Handling better supported channel binding types for SSL implementations](https://commitfest.postgresql.org/patch/1491/) (cf, Rejected, 2018-08-06) - judged not relevant
- P15: [Fix data types where equality comparison is not intuitive, e.g. box](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Data_Types) (do_not_want, Not worth doing, date unknown) - used in the review
- P16: [doc: PQexecParams binary handling example for REAL data type](https://commitfest.postgresql.org/patch/3568/) (cf, Returned with feedback, 2023-07-26) - judged not relevant
- P17: [Revert "Reject degenerate SPLIT PARTITION with DEFAULT partition"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0392fb900eb89f52988cccd33046443c39c70d1c) (revert, Reverted, 2026-05-20) - judged not relevant
- P18: [Revert "Reject concurrent sequence refreshes".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dde9a87d4d02a67a5bd459045f03a6f467a00a80) (revert, Reverted, 2026-07-18) - judged not relevant
- P19: [better handling of XPath data types \[3\] \[4\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P20: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant
- P21: [Autovacuum: vacuum tables in descending order of xid_age](https://commitfest.postgresql.org/patch/2364/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P22: [KNN for B-tree](https://commitfest.postgresql.org/patch/1804/) (cf, Returned with feedback, 2020-03-16) - judged not relevant
- P23: [Join Correlation Name](https://commitfest.postgresql.org/patch/2322/) (cf, Withdrawn, 2019-11-01) - judged not relevant
- P24: [KNN-btree](https://commitfest.postgresql.org/patch/966/) (cf, Returned with feedback, 2017-03-09) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 132 s; cost reported by the CLI: $1.97
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 83 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:20:24.382Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
