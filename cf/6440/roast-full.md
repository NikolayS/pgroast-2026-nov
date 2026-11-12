# Design pre-review: tablecmds: fix bug where index rebuild loses replica identity on partitions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6440](https://commitfest.postgresql.org/patch/6440/) (Needs review); authors: Chao Li (chaol)
- Thread: [tablecmds: fix bug where index rebuild loses replica identity on partitions](https://www.postgresql.org/message-id/flat/DB533C25-C6BA-4C0F-8046-96168E9CDD72%40gmail.com), started 2026-01-27
- Messages read: 45 of 45
- Patch prose read: v17-0001-Preserve-index-per-column-statistics-targets-acr.patch, v17-0002-Preserve-descendant-partition-index-properties-a.patch, v1-0001-tablecmds-fix-bug-where-index-rebuild-loses-repli.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No objection survived against the goal: the loss of replica identity and other partition index properties is real and reproducible. Three design points are open at level 2. The series does not say which cases still need save-and-restore if the rebuild-avoidance patch from message #45 lands (maintenance-fit-2). It names no target branches (replication-ondisk-1). It does not state which properties the rebuild keeps (standards-compat-2, maintenance-fit-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 45 messages and no code were read, and the v19 prose was not shown. This review could not check whether v19 covers every lost property, whether the reserved-name list avoids all name clashes, or whether restored tablespaces pass the usual privilege check.

## The proposal, restated

**Problem.** When `alter table ... alter column type` or `set expression` rebuilds a partitioned index, Postgres drops the descendant partition indexes and recreates them from the parent definition. Replica identity, cluster-on markings, custom names, comments, statistics targets, reloptions, tablespaces and extension dependencies on those indexes are silently lost. Users of logical replication and anyone who customized partition indexes are affected.

**Proposed design.** Before the old index hierarchy is dropped, the patch saves the properties of each descendant index, carries them in the IndexStmt through DefineIndex() recursion, and restores them on the replacement index and constraint by direct catalog updates with post-alter hooks. The latest series (v19) has three patches: keep `depends on extension` links, keep per-column statistics targets for all indexes, and keep descendant partition index properties, with a reserved-name list to avoid name clashes.

**User-visible surface.**
- `alter table ... alter column type` and `set expression` now keep custom names of descendant partition indexes.
- Replica identity and cluster-on (`repack ... using index`) markings on partition indexes survive the rebuild.
- Index comments, constraint comments, reloptions and per-index tablespaces of partition indexes survive the rebuild.
- Per-column statistics targets on any index (`alter index ... alter column n set statistics`) survive the rebuild.
- `alter index ... depends on extension` links survive the rebuild.

**What the author claims.**
- The loss of replica identity on partitions leaves logical replication misconfigured.
- The bug is easy to reproduce and old, so the author first saw it as a back-patch candidate.
- Restructuring the alter table drop and rebuild order, as Alberto Piai suggested, needs refactoring beyond a bug fix.
- check-world passes with v15.

**Thread state.** Sami Imseih now co-develops the series and posted v19 after Zsolt Parragi found a name collision. Melanie Plageman doubts a back-patch and asks whether the intended semantics are defined, and Álvaro Rodríguez proposes merging with his patch that avoids rebuilding leaf indexes.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-01-27; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Two competing mechanisms for the same rebuild path

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

0003 is committed, and later the rebuild-avoidance patch from message #45 is committed as well. Leaf indexes are then no longer dropped in most cases, but the save-and-restore code stays and runs only in rare paths. Later changes to tablecmds.c must keep both mechanisms correct, and the rare path gets little testing.

Message #45 reports that a patch which reorders processing and avoids rebuilding leaf indexes covers a subset of what 0003 does, and that 0003's tests pass with it. Message #15 and message #18 traced the loss to the parent drop cascading to the partition indexes before they are remembered. The author's answer in message #19 is that fixing that order needs refactoring beyond a bug fix. That answer carries less weight now that the series is no longer a narrow bug fix and another patch does that reordering.

Thread: [#15 ("Alberto Piai", 2026-07-22)](https://www.postgresql.org/message-id/DK5H6VM4U4J4.C8APP2ZH2CAT%40gmail.com), [#18 ("Alberto Piai", 2026-07-28)](https://www.postgresql.org/message-id/DKA06URPT1BS.VMPKMBIU9JQQ%40gmail.com), [#19 (Chao Li, 2026-07-28)](https://www.postgresql.org/message-id/EFBAC69A-E37C-4A89-AE6F-28F2A1C5F46F%40gmail.com), [#25 ("Alberto Piai", 2026-07-31)](https://www.postgresql.org/message-id/DKCPTOUWSN8G.ZOEE18C6M6QE%40gmail.com), [#45 (Álvaro Rodríguez, 2026-10-06)](https://www.postgresql.org/message-id/CA%2BC_kKWVb5PrrGJM5Bd4rPsf-V%3Dh20xoeCHMtPy3WCYy4%3DYwUQ%40mail.gmail.com)

**Question:** If the rebuild-avoidance patch lands, which cases still need 0003's save-and-restore code, and should that code be designed only for those cases?

Counter-check: held. Message #25 accepted save-and-restore only because reordering looked too large, and message #45 now reports a patch that does the reordering and passes 0003's tests. The thread has no answer on which cases would still need 0003.

### 2. Back-patching changes index names and the IndexStmt struct in minor releases

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API, WAL / replication / standby · `replication-ondisk-1`

A user's upgrade script runs `alter column type` and then `alter table child replica identity using index child_id_val_idx`, the name that rebuild generates today. After a minor update, the custom index name survives, so the script fails. A publisher and a subscriber on different minor releases end the same DDL with different index names and replica identity markings.

The CommitFest entry targets stable, and message #22 still treats the fix as a back-patch candidate. Message #28 argues against back-patching because users likely re-apply these properties in scripts. v17 and v19 add fields to IndexStmt in parsenodes.h, a node that extensions read through ProcessUtility hooks. Today the series also preserves names, comments, tablespaces and extension dependencies, which goes well beyond the replica identity bug that justified a back-patch in message #1.

Thread: [#1 (Chao Li, 2026-01-27)](https://www.postgresql.org/message-id/DB533C25-C6BA-4C0F-8046-96168E9CDD72%40gmail.com), [#7 (Chao Li, 2026-03-19)](https://www.postgresql.org/message-id/260E4616-43B7-4C24-BF53-352DC8DEB7AA%40gmail.com), [#22 (Chao Li, 2026-07-29)](https://www.postgresql.org/message-id/3A41A063-A024-4DC4-816B-9BD516079B4B%40gmail.com), [#28 (Melanie Plageman, 2026-08-24)](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com)

**Question:** Will the series target master only, and if any part is back-patched, which part avoids both the IndexStmt change and the change of index names?

Counter-check: held. The CommitFest entry still says stable, and no message after #28 names the target branches. Kept custom names and new IndexStmt fields are both visible changes in a minor release; the publisher and subscriber sentence adds little, because users run DDL on each side themselves.

### 3. The contract for which properties survive the rebuild is not written down

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A developer adds a new per-index property in a later release, and nothing in the docs says alter column type must keep it. The new property is lost on partition rebuilds, and a user only notices it after replication or a planner choice changes.

Reviewers found a new lost property in almost every round: name, cluster-on, comments, stats targets, reloptions, tablespace, constraint comments, extension dependencies (messages #16, #17, #28, #34, #40). Message #28 asks whether the intended semantics are defined at all, and message #31 defers that question. The patch prose shows no documentation hunk for `alter table` that states which properties are kept. The catalog-diff test from message #28 helps, but it does not tell users what to expect.

Thread: [#16 (Sami Imseih, 2026-07-27)](https://www.postgresql.org/message-id/CAA5RZ0s6P1q38rLdBbKCB6X4vXs-2uuUZOARkc03b-K1-uHQag%40mail.gmail.com), [#17 (Sami Imseih, 2026-07-27)](https://www.postgresql.org/message-id/CAA5RZ0tq2Oj6Vkjt06phsDWpywsS%3DVkBSV7n6LC%2B%2BpR5LLNU-A%40mail.gmail.com), [#28 (Melanie Plageman, 2026-08-24)](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com), [#31 (Chao Li, 2026-08-26)](https://www.postgresql.org/message-id/D088FD51-422F-436E-8650-85916A7D122D%40gmail.com), [#34 (Zsolt Parragi, 2026-08-26)](https://www.postgresql.org/message-id/CAN4CZFNQHDgaUkQgtNrxUz96-RqcvhwxUNEAeSV7TAckG5TEng%40mail.gmail.com), [#40 (Sami Imseih, 2026-08-27)](https://www.postgresql.org/message-id/CAN12%2BYKriFqdTU3uhKnRMaKkWmm4ZNfzm0d6eT9PFFGFvxojMw%40mail.gmail.com)

**Question:** Can the series add `alter table` documentation that lists which index properties alter column type keeps, and say how a new property gets added to that list?

Counter-check: held. Message #28 asks whether the intended semantics are defined, and message #31 defers that question. No later message states which properties `alter column type` keeps, so users with follow-up scripts cannot know what to expect.

### 4. Hand-kept property list fails silently for future index properties

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), catalog changes · `maintenance-fit-1`

A later release adds a new per-index property, stored in pg_class, pg_index, pg_description or pg_depend. Its author does not know about the save-and-restore list in DefineIndex(). After the next alter column type on a partitioned table, that property disappears from every descendant index, and users notice only when replication or maintenance behaves differently.

The design restores a fixed set of properties, and each review round found more that were missing: comments, statistics targets and reloptions (message #28), tablespace (message #31), intermediate indexes and constraint comments (message #34), and extension dependencies (message #40). Message #28 proposed a catalog-diff test that forces new properties to be listed, but the material does not show that v19 still has it. Such a test would not see properties stored in pg_description or pg_depend.

Thread: [#28 (Melanie Plageman, 2026-08-24)](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com), [#31 (Chao Li, 2026-08-26)](https://www.postgresql.org/message-id/D088FD51-422F-436E-8650-85916A7D122D%40gmail.com), [#34 (Zsolt Parragi, 2026-08-26)](https://www.postgresql.org/message-id/CAN4CZFNQHDgaUkQgtNrxUz96-RqcvhwxUNEAeSV7TAckG5TEng%40mail.gmail.com), [#40 (Sami Imseih, 2026-08-27)](https://www.postgresql.org/message-id/CAN12%2BYKriFqdTU3uhKnRMaKkWmm4ZNfzm0d6eT9PFFGFvxojMw%40mail.gmail.com)

**Question:** Which test in v19 fails when someone adds a new index property, in any catalog, that the rebuild does not carry over?

Counter-check: held. Message #28 cites index_concurrently_swap() as existing practice for hand-kept transfers, which lowers the weight. But reviewers found missing properties in messages #31, #34 and #40, and nothing shows the catalog-diff test is in v19 or covers pg_description and pg_depend.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The series restores replica identity markings on partition indexes, which logical replication depends on. |
| wire protocol | no |  |
| catalog changes | **yes** | The patch restores properties by direct catalog updates on the rebuilt indexes and constraints; it adds no new catalog. |
| ABI / extension API | **yes** | The patches add fields to IndexStmt in parsenodes.h, a node that extensions read through ProcessUtility hooks. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-2: The contract for which properties survive the rebuild is not written down |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch restores per-index tablespaces and calls post-alter hooks; the tablespace privilege check on restore is unverified without the code. |
| backpatchability | **yes** | The CommitFest entry targets stable, while message #28 argues against a back-patch. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A hand-kept list of index properties in tablecmds.c and DefineIndex() must grow with every new index property. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. If the rebuild-avoidance patch from message #45 lands, which cases still need 0003's save-and-restore code?
2. Does the series target master only, and if not, which part can be back-patched without changing IndexStmt or the resulting index names?
3. Which index properties does `alter column type` keep after this series, and where will the `alter table` documentation say so?
4. Which test in v19 fails when a new index property, stored in any catalog, is not carried over by the rebuild?
5. How should scripts that re-apply these properties after `alter column type`, using the regenerated index names, behave after this change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The owner must know the `alter table` rebuild path in tablecmds.c and the DefineIndex() recursion over partitions. Melanie Plageman posts patches and discusses committing in message #28.
- **Liability:** The owner signs for the list of preserved properties in every later release, including each new index property. If back-patched, the owner also signs for changed index names and the IndexStmt layout in minor releases.
- **Author follow-up:** The author kept the entry alive through five CommitFests and many versions, and Sami Imseih now co-develops the series and posted v19.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Users of partitioned tables with logical replication or customized partition indexes lose settings silently, so the goal is worth reviewer time. The main patch is about 37 KiB in tablecmds.c and indexcmds.c, and its scope depends on the unsettled overlap with the rebuild-avoidance patch and on the target branches.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Back-patching changes alter column type results for existing scripts. Refuted (not specific): Duplicate of replication-ondisk-1, which makes the same script-breakage point and adds the IndexStmt layout change. The back-patch remark it cites is in message #22, not message #21.
- `maintenance-fit-3` Stable-branch target conflicts with new IndexStmt fields and open semantics. Refuted (not specific): Duplicate of replication-ondisk-1. Its added claim that no committer is engaged conflicts with message #28, where Melanie Plageman posts patches and writes about committing the series.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P16] makes replica identity recurse to partitions, a different idea, and no reason is recorded for it. Messages #28 and #45 point to related open work (Zsolt's statistics-target patch, a semantics thread, Álvaro's rebuild-avoidance patch), but the thread reports none of it as rejected.
- standards-compat: 2 objection(s). No new syntax, GUC or catalog object is added, so pg_dump round-trip and psql conventions are not affected.
- replication-ondisk: 1 objection(s). From this angle, the series changes no on-disk format, WAL record or wire protocol. The catalog writes set properties the catalog already stores. The objection above only applies if the series is back-patched.
- security: 0 objection(s). The series has no privilege or exposure aspect. Only the table owner can run `alter column type`, and it restores properties that an owner of the same objects set before, so no role gains an ability it lacked. Without the code, two points stay open: whether restored descendant tablespaces still pass the usual CREATE privilege check on the tablespace, and whether the post-alter hooks added in message #36 fire for every restored property.
- maintenance-fit: 3 objection(s). The direct catalog updates that skipped post-alter hooks were raised in message #29 and message #35 and addressed in message #36, so they are not raised here. Hot-path cost does not apply, because the code runs only during DDL.
- already-exists: 0 objection(s). Postgres has no existing feature that keeps these properties through the rebuild. The only workaround is to re-apply each one by hand afterwards, with REPLICA IDENTITY USING INDEX, COMMENT ON, ALTER INDEX RENAME, SET STATISTICS, SET (...), SET TABLESPACE and DEPENDS ON EXTENSION. That works only if the user already knows the loss happens, and the root index keeps its properties while descendants lose them. Saving and restoring the properties follows index_concurrently_swap() (message #28). Rebuilding per partition instead was weighed and dropped by the reviewer who proposed it (message #25). The overlap with the leaf rebuild-avoidance patch (message #45) is with another pending patch, not with an existing feature, so this angle does not cover it.
- demand: 0 objection(s). The material shows a real, reproducible silent loss of replica identity and other index settings, which affects logical replication users. Several independent developers reproduced it and found more cases, and the cost is limited to the alter column type rebuild path, so a demand objection would not hold.

### Precedents retrieved

- P1: [DROP INDEX CONCURRENTLY on partitioned index](https://commitfest.postgresql.org/patch/2805/) (cf, Withdrawn, 2021-07-24) - judged not relevant; reason summarised from its thread
- P2: [CREATE INDEX CONCURRENTLY for partitioned tables](https://commitfest.postgresql.org/patch/5004/) (cf, Withdrawn, 2025-04-08) - judged not relevant; reason summarised from its thread
- P3: [CREATE INDEX CONCURRENTLY on partitioned table](https://commitfest.postgresql.org/patch/2815/) (cf, Withdrawn, 2023-09-18) - judged not relevant; reason summarised from its thread
- P4: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P5: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P6: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant; reason summarised from its thread
- P7: [Add support for ALTER INDEX .. ALTER \[COLUMN\] col_num {SET,RESET}](https://commitfest.postgresql.org/patch/3382/) (cf, Withdrawn, 2021-11-01) - judged not relevant; reason summarised from its thread
- P8: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant; reason summarised from its thread
- P9: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P10: [Revert support for ALTER TABLE ... MERGE/SPLIT PARTITION(S) commands](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=3890d90c1508125729ed20038d90513694fc3a7b) (revert, Reverted, 2024-08-24) - judged not relevant
- P11: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P12: [Introduce the ability to enable/disable indexes using ALTER INDEX](https://commitfest.postgresql.org/patch/5274/) (cf, Withdrawn, 2025-09-08) - judged not relevant
- P13: [Exclude partition constaint checking in query execution plan for partitioned table queries](https://commitfest.postgresql.org/patch/1264/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant
- P15: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant
- P16: [propagating replica identity to partitions](https://commitfest.postgresql.org/patch/2044/) (cf, Returned with feedback, 2019-09-11) - used in the review
- P17: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/3984/) (cf, Returned with feedback, 2023-07-10) - judged not relevant
- P18: [Use indexes on the subscriber when REPLICA IDENTITY is full on the publisher](https://commitfest.postgresql.org/patch/3926/) (cf, Returned with feedback, 2022-10-20) - judged not relevant
- P19: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - judged not relevant
- P20: [Split index and table statistics into different types of stats](https://commitfest.postgresql.org/patch/4792/) (cf, Returned with feedback, 2024-10-02) - judged not relevant
- P21: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant
- P22: [Implement type input functions for extended statistics types](https://commitfest.postgresql.org/patch/4114/) (cf, Withdrawn, 2023-03-01) - judged not relevant
- P23: [Revert changes in HOT handling of BRIN indexes](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e3fcca0d0d2414f3a50d6fd40eddf48b7df81475) (revert, Reverted, 2022-06-16) - judged not relevant
- P24: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 154 s; cost reported by the CLI: $3.86
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 0 from cache, 8 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 77 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:59:11.727Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
