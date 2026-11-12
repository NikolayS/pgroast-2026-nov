# Design pre-review: tablecmds: fix bug where index rebuild loses replica identity on partitions

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6440](https://commitfest.postgresql.org/patch/6440/); [thread](https://www.postgresql.org/message-id/flat/DB533C25-C6BA-4C0F-8046-96168E9CDD72%40gmail.com), 45 of 45 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No objection survived against the goal: the loss of replica identity and other partition index properties is real and reproducible. Three design points are open at level 2. The series does not say which cases still need save-and-restore if the rebuild-avoidance patch from message #45 lands (maintenance-fit-2). It names no target branches (replication-ondisk-1). It does not state which properties the rebuild keeps (standards-compat-2, maintenance-fit-1).

Not checked: Only 3 of 45 messages and no code were read, and the v19 prose was not shown. This review could not check whether v19 covers every lost property, whether the reserved-name list avoids all name clashes, or whether restored tablespaces pass the usual privilege check.

## Prior attempts

None found before 2026-01-27.

## Objections a reviewer is likely to raise

### 1. Two competing mechanisms for the same rebuild path

0003 is committed, and later the rebuild-avoidance patch from message #45 is committed as well. Leaf indexes are then no longer dropped in most cases, but the save-and-restore code stays and runs only in rare paths. Later changes to tablecmds.c must keep both mechanisms correct, and the rare path gets little testing.

Message #45 reports that a patch which reorders processing and avoids rebuilding leaf indexes covers a subset of what 0003 does, and that 0003's tests pass with it. Message #15 and message #18 traced the loss to the parent drop cascading to the partition indexes before they are remembered. The author's answer in message #19 is that fixing that order needs refactoring beyond a bug fix. That answer carries less weight now that the series is no longer a narrow bug fix and another patch does that reordering.

**Question:** If the rebuild-avoidance patch lands, which cases still need 0003's save-and-restore code, and should that code be designed only for those cases?

<sub>fixable, level 2, design; messages [#15](https://www.postgresql.org/message-id/DK5H6VM4U4J4.C8APP2ZH2CAT%40gmail.com), [#18](https://www.postgresql.org/message-id/DKA06URPT1BS.VMPKMBIU9JQQ%40gmail.com), [#19](https://www.postgresql.org/message-id/EFBAC69A-E37C-4A89-AE6F-28F2A1C5F46F%40gmail.com), [#25](https://www.postgresql.org/message-id/DKCPTOUWSN8G.ZOEE18C6M6QE%40gmail.com), [#45](https://www.postgresql.org/message-id/CA%2BC_kKWVb5PrrGJM5Bd4rPsf-V%3Dh20xoeCHMtPy3WCYy4%3DYwUQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 2. Back-patching changes index names and the IndexStmt struct in minor releases

A user's upgrade script runs `alter column type` and then `alter table child replica identity using index child_id_val_idx`, the name that rebuild generates today. After a minor update, the custom index name survives, so the script fails. A publisher and a subscriber on different minor releases end the same DDL with different index names and replica identity markings.

The CommitFest entry targets stable, and message #22 still treats the fix as a back-patch candidate. Message #28 argues against back-patching because users likely re-apply these properties in scripts. v17 and v19 add fields to IndexStmt in parsenodes.h, a node that extensions read through ProcessUtility hooks. Today the series also preserves names, comments, tablespaces and extension dependencies, which goes well beyond the replica identity bug that justified a back-patch in message #1.

**Question:** Will the series target master only, and if any part is back-patched, which part avoids both the IndexStmt change and the change of index names?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/DB533C25-C6BA-4C0F-8046-96168E9CDD72%40gmail.com), [#7](https://www.postgresql.org/message-id/260E4616-43B7-4C24-BF53-352DC8DEB7AA%40gmail.com), [#22](https://www.postgresql.org/message-id/3A41A063-A024-4DC4-816B-9BD516079B4B%40gmail.com), [#28](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 3. The contract for which properties survive the rebuild is not written down

A developer adds a new per-index property in a later release, and nothing in the docs says alter column type must keep it. The new property is lost on partition rebuilds, and a user only notices it after replication or a planner choice changes.

Reviewers found a new lost property in almost every round: name, cluster-on, comments, stats targets, reloptions, tablespace, constraint comments, extension dependencies (messages #16, #17, #28, #34, #40). Message #28 asks whether the intended semantics are defined at all, and message #31 defers that question. The patch prose shows no documentation hunk for `alter table` that states which properties are kept. The catalog-diff test from message #28 helps, but it does not tell users what to expect.

**Question:** Can the series add `alter table` documentation that lists which index properties alter column type keeps, and say how a new property gets added to that list?

<sub>fixable, level 2, design; messages [#16](https://www.postgresql.org/message-id/CAA5RZ0s6P1q38rLdBbKCB6X4vXs-2uuUZOARkc03b-K1-uHQag%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAA5RZ0tq2Oj6Vkjt06phsDWpywsS%3DVkBSV7n6LC%2B%2BpR5LLNU-A%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/D088FD51-422F-436E-8650-85916A7D122D%40gmail.com), [#34](https://www.postgresql.org/message-id/CAN4CZFNQHDgaUkQgtNrxUz96-RqcvhwxUNEAeSV7TAckG5TEng%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/CAN12%2BYKriFqdTU3uhKnRMaKkWmm4ZNfzm0d6eT9PFFGFvxojMw%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Hand-kept property list fails silently for future index properties

A later release adds a new per-index property, stored in pg_class, pg_index, pg_description or pg_depend. Its author does not know about the save-and-restore list in DefineIndex(). After the next alter column type on a partitioned table, that property disappears from every descendant index, and users notice only when replication or maintenance behaves differently.

The design restores a fixed set of properties, and each review round found more that were missing: comments, statistics targets and reloptions (message #28), tablespace (message #31), intermediate indexes and constraint comments (message #34), and extension dependencies (message #40). Message #28 proposed a catalog-diff test that forces new properties to be listed, but the material does not show that v19 still has it. Such a test would not see properties stored in pg_description or pg_depend.

**Question:** Which test in v19 fails when someone adds a new index property, in any catalog, that the rebuild does not carry over?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#28](https://www.postgresql.org/message-id/CAAKRu_ajK0QhmOA3N2-QynT7hXx-VVV99RTXrae3sc51we2TSA%40mail.gmail.com), [#31](https://www.postgresql.org/message-id/D088FD51-422F-436E-8650-85916A7D122D%40gmail.com), [#34](https://www.postgresql.org/message-id/CAN4CZFNQHDgaUkQgtNrxUz96-RqcvhwxUNEAeSV7TAckG5TEng%40mail.gmail.com), [#40](https://www.postgresql.org/message-id/CAN12%2BYKriFqdTU3uhKnRMaKkWmm4ZNfzm0d6eT9PFFGFvxojMw%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. If the rebuild-avoidance patch from message #45 lands, which cases still need 0003's save-and-restore code?
2. Does the series target master only, and if not, which part can be back-patched without changing IndexStmt or the resulting index names?
3. Which index properties does `alter column type` keep after this series, and where will the `alter table` documentation say so?
4. Which test in v19 fails when a new index property, stored in any catalog, is not carried over by the rebuild?
5. How should scripts that re-apply these properties after `alter column type`, using the regenerated index names, behave after this change?

## Touch points

- **WAL / replication / standby:** The series restores replica identity markings on partition indexes, which logical replication depends on.
- **catalog changes:** The patch restores properties by direct catalog updates on the rebuilt indexes and constraints; it adds no new catalog.
- **ABI / extension API:** The patches add fields to IndexStmt in parsenodes.h, a node that extensions read through ProcessUtility hooks.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: The contract for which properties survive the rebuild is not written down
- **security / privileges:** The patch restores per-index tablespaces and calls post-alter hooks; the tablespace privilege check on restore is unverified without the code.
- **backpatchability:** The CommitFest entry targets stable, while message #28 argues against a back-patch.
- **maintenance burden ("who maintains this in 10 years"):** A hand-kept list of index properties in tablecmds.c and DefineIndex() must grow with every new index property.

## Who would commit it, and when

- Owner: qualified committer engaged. The owner must know the `alter table` rebuild path in tablecmds.c and the DefineIndex() recursion over partitions. Melanie Plageman posts patches and discusses committing in message #28.
- Priority: fix design first; audience a sizeable minority; review cost medium. Users of partitioned tables with logical replication or customized partition indexes lose settings silently, so the goal is worth reviewer time. The main patch is about 37 KiB in tablecmds.c and indexcmds.c, and its scope depends on the unsettled overlap with the rebuild-avoidance patch and on the target branches.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
