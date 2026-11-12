# Design pre-review: tablecmds: fix Deadlock caused by Attach Partition

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6442](https://commitfest.postgresql.org/patch/6442/) (Needs review); authors: Chao Li (chaol)
- Thread: [tablecmds: Deadlock caused by Attach Partition](https://www.postgresql.org/message-id/flat/CFACA0EB-7E6F-4FAA-9ACE-1FC2226D7482%40gmail.com), started 2026-01-30
- Messages read: 16 of 16
- Patch prose read: v5-0001-Prevent-deadlock-between-ALTER-TABLE-recursion-an.patch, v5-0002-Prevent-deadlocks-during-partitioned-index-attach.patch, v5-0003-Add-isolation-tests-for-ATTACH-PARTITION-lock-ord.patch, v1-0001-Prevent-deadlock-between-ALTER-TABLE-recursion-an.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is sound and 0001 adds only AccessShareLock on ancestors, which master already takes with a cold cache. The design of 0002 is open: standards-compat-1 and replication-ondisk-1 show it takes AccessExclusiveLock on ancestor tables and indexes for every multi-level index ATTACH, and no committer has accepted that cost. maintenance-fit-2 shows other bottom-up paths remain, and the author now holds the patch for a general fix.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check whether the callback releases ancestor locks correctly across lookup retries, or how it handles a target reparented before it is locked. It also could not check the order of permission checks and ancestor locks.

## The proposal, restated

**Problem.** A recursive ALTER TABLE on a partitioned table, such as ALTER COLUMN SET DEFAULT, locks the tree top-down. ATTACH PARTITION on a mid-level table locks the target and then its ancestors bottom-up in generate_partition_qual(). Run concurrently, the two deadlock, and the author reproduces this on master with deep partition chains.

**Proposed design.** RangeVarCallbackForAttachPartition() locks all ancestor tables from root to parent before the target is locked, follows pending-detach links, and tracks the locks across lookup retries. A second patch extends this to ALTER INDEX ... ATTACH PARTITION and takes AccessExclusiveLock on ancestor indexes and their tables. A third patch adds isolation tests.

**User-visible surface.**
- ALTER TABLE ... ATTACH PARTITION always takes AccessShareLock on every ancestor table, even when the partition constraint is cached.
- ALTER INDEX ... ATTACH PARTITION always takes AccessExclusiveLock on ancestor indexes and their tables, even when no validation follows.
- New isolation tests partition-attach-lock-order and partition-index-attach-lock-order.

**What the author claims.**
- The deadlock is reliably reproducible on master with deep partition chains.
- Lock order now follows the partition hierarchy and does not depend on relative OIDs.
- DML directly on a partition has a known deadlock of the same kind and is out of scope.
- The author later says many more bottom-up locking paths exist, such as DROP NOT NULL and pg_get_partition_constraintdef().
- The author says these deadlocks are rare, retry usually works, and the patch is not urgent.

**Thread state.** Surya Poondla reviewed v1 to v5 and asked Álvaro Herrera to decide whether the extra AccessExclusiveLock in the index fix is acceptable; the CF status went back to Needs review. The author then said he would hold the patch and look for a general fix, which he has not yet designed.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-01-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Index ATTACH now takes AccessExclusiveLock on the root table every time

fixable · level 2, rubric item 4 · design, performance on the hot path, backpatchability · `standards-compat-1`

A DBA attaches leaf indexes one by one on a busy multi-level tree, and most of these attaches validate nothing. With 0002, each attach takes AccessExclusiveLock on the root table and the ancestor indexes. Every query on the root then waits until the attach commits, and the DBA sees this only as an outage in production.

Message #14 measured this. Master locks no ancestors when the parent index stays invalid, and v5 takes AccessExclusiveLock on root, mid1 and their indexes. The same message gives the reason a weaker lock does not work, and asks a committer to accept the trade-off, which no committer has done. The entry is tagged Bugfix, so this lock change could also reach minor releases, and the patch prose touches no documentation of the new lock set.

Thread: [#14 (surya poondla, 2026-10-27)](https://www.postgresql.org/message-id/CAOVWO5pmhdGfo7mvjheg1kryjMXNumDOiN9%3Dio2nthO%3DomRRNA%40mail.gmail.com)

**Question:** Can the deadlock be avoided without AccessExclusiveLock on the root when the attach will not validate any ancestor index, and if not, will the documentation state the new lock?

Counter-check: held. Message #14 measured the new AccessExclusiveLock on ancestors of a mid-level index, and no committer answered the trade-off question. The backpatch remark is speculation, since the entry targets version 19, and should not carry weight.

### 2. New AccessExclusiveLock in index ATTACH also blocks hot standby readers

fixable · level 2, rubric item 8 · WAL / replication / standby, performance on the hot path · `replication-ondisk-1`

A DBA attaches a leaf index in a large partition tree where the parent index stays invalid. The primary WAL-logs the new AccessExclusiveLock on the root table and its index, and the standby replays it. Reporting queries on the root at the standby then wait or are cancelled after max_standby_streaming_delay, although master took no ancestor locks for this ATTACH.

Message #14 shows that v5-0002 takes AccessExclusiveLock on root, mid1, root_idx and mid1_idx even when nothing is validated, where master takes none. Postgres WAL-logs AccessExclusiveLock so hot standby can replay it, so this cost reaches every standby. The discussion in message #14 covers only blocking on the primary, and that question is still open. A committer would likely want the standby effect weighed in the same trade-off.

Thread: [#14 (surya poondla, 2026-10-27)](https://www.postgresql.org/message-id/CAOVWO5pmhdGfo7mvjheg1kryjMXNumDOiN9%3Dio2nthO%3DomRRNA%40mail.gmail.com)

**Question:** How does the extra AccessExclusiveLock affect queries on hot standbys when no validation follows, and can the non-cascading case avoid it?

Counter-check: held. Postgres WAL-logs AccessExclusiveLock and standbys replay it, so standby queries on the root can wait or be cancelled. Message #14 weighs only blocking on the primary, so this cost is new to the trade-off.

### 3. Point fix leaves the bottom-up locking pattern open for future callers

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A later patch adds a new caller of generate_partition_qual() or another child-then-parent walk, and does not call lock_partition_ancestors() first. No test or assertion fails. Users see occasional deadlocks against recursive ALTER TABLE in production, and a new thread starts to hunt for the cause.

After v5, the thread found the same inversion in pg_get_partition_constraintdef(), which psql \d+ uses (message #15), and in DROP NOT NULL (message #16). The author now says these should not be fixed one at a time and is holding the patch. Committing 0001 and 0002 as they stand adds a convention that every bottom-up path must follow, with nothing to enforce it, and a later general redesign may have to undo or rework them.

Thread: [#12 (Álvaro Herrera, 2026-10-25)](https://www.postgresql.org/message-id/at3MCB1EIOHMjh6K%40alvherre.pgsql), [#15 (shihao zhong, 2026-10-28)](https://www.postgresql.org/message-id/CAGRkXqTSqk-4ej3_q_frxGKvf-A8CcgCH_bvvEnvPuzaYe5u-w%40mail.gmail.com), [#16 (Chao Li, 2026-10-29)](https://www.postgresql.org/message-id/1CE846D3-C504-436B-8F59-E9296CA22A4A%40gmail.com)

**Question:** Which rule, enforced in code or by tests, keeps the next caller that walks a partition up to its ancestors from bringing the deadlock back?

Counter-check: held. Messages #15 and #16 confirm more bottom-up paths, and the author now prefers a general fix. The patch adds no assertion or comment that makes a new caller fail loudly; message #15 only suggests a comment.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | 0002 adds AccessExclusiveLock on ancestor tables and indexes, which Postgres WAL-logs and hot standbys replay. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0001 adds an exported function to src/include/catalog/partition.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix, but it changes the lock set of ATTACH PARTITION, which is a behaviour change for stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every path that walks from a partition up to its ancestors must pre-lock them, and nothing enforces this. |
| performance on the hot path | **yes** | Every ATTACH in a multi-level tree takes ancestor locks, and index ATTACH takes AccessExclusiveLock on the root table. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can ALTER INDEX ... ATTACH PARTITION avoid AccessExclusiveLock on ancestor tables and indexes when the attach validates no ancestor index?
2. How does the new AccessExclusiveLock on the root affect queries on hot standbys, where replay can make them wait or cancel them?
3. Which assertion, test or code rule makes a new caller of generate_partition_qual() fail loudly if it locks ancestors after the partition?
4. Should 0001 go ahead alone now, and will the general fix from message #16 keep the ancestor lock set it puts in place?
5. What happens when the target is reparented between the ancestor lookup and the lock on the target, which the patch only comments on?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows partition locking in tablecmds.c and the index validation path. Message #14 asked for a committer decision on the index lock and got no reply.
- **Liability:** The owner signs for a wider lock set on every multi-level ATTACH, including AccessExclusiveLock on root tables for index ATTACH, and for any blocking or standby cancellation reports that follow. The pre-lock convention also needs care in every later patch that walks up a partition tree.
- **Author follow-up:** The author answered reviews through v5, then said in message #16 that he holds the patch to look for a general fix that is not yet designed.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The deadlock needs a multi-level partition tree and concurrent DDL, and the author calls it rare and retryable. The patches are small, but the index lock trade-off has no committer answer and the author is holding the patch for a general fix.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Narrow fix leaves the same lock inversion open on other known paths. Refuted (not specific): Duplicate of maintenance-fit-2. That objection states the same missing rule for future generate_partition_qual() callers, without mixing in deadlocks that exist on master today.
- `standards-compat-2` Partial fix changes the lock contract while psql \d+ still deadlocks. Refuted (no failure scenario): The \d+ deadlock exists on master, so the patch does not cause it. Lock sets are not a stable contract across major releases, and 0001 takes the same AccessShareLock set as master with a cold cache.
- `security-1` Ancestor locks may be taken before ownership checks and by non-owners. Refuted (no failure scenario): Master already lets a partition owner lock ancestors it does not own: DROP of a partition locks the parent, and cascading index validation locks ancestors. The claim that locks precede the ownership check has no support in the material.
- `maintenance-fit-1` Index ATTACH would block the root table on every call. Refuted (not specific): Duplicate of standards-compat-1. This version also overstates the scope: with one partition level the target index has no ancestors, so 0002 adds no lock, and dropping 0002 leaves 0001 usable.
- `demand-1` Index ATTACH locks the root table for every user to avoid a rare deadlock. Refuted (not specific): Duplicate of standards-compat-1, which states the same cost against the same rare deadlock and names the multi-level case.
- `demand-2` No field reports, and this fixes one of many bottom-up paths. Refuted (not specific): Duplicate of maintenance-fit-2 for the one-of-many-paths point. [P6] concerns recursion policy for SET LOGGED and does not apply, and a three-level isolation test in message #4 reproduces the deadlock without a 1000-level chain.

### Review angles

- prior-attempts: 1 objection(s). No precedent with the same idea was found. Message #4 points to a 2021 discussion of the DML variant, which the thread says was accepted as rare and does not cover ATTACH.
- standards-compat: 2 objection(s). No new syntax, GUC, catalog object or pg_dump output is proposed, so standard, grammar and dump round-trip concerns do not apply.
- replication-ondisk: 1 objection(s). The patches change only the lock order and lock set. They add no catalog, WAL record, on-disk or wire protocol change, and the new partition.h function does not break ABI. The one cross-node effect is replay of the new AccessExclusiveLock on standbys.
- security: 1 objection(s). Only the lock-before-permission ordering and the cross-owner lock reach are security concerns. The table ATTACH path takes AccessShareLock, which master already takes on a cold cache.
- maintenance-fit: 2 objection(s). 0001 alone copies existing practice from RangeVarCallbackForDropRelation() and only adds AccessShareLock on ancestors, so I raise no separate objection to it.
- already-exists: 0 objection(s). This is a bug fix for a lock-order deadlock in core DDL, and nothing outside core can fix it. An application can work around it by running `lock table` on each ancestor, root first, in the same transaction before ATTACH, or by retrying after the deadlock error, but that leaves core behaviour unchanged. The design reuses the pattern of RangeVarCallbackForDropRelation(), so it does not add a second mechanism for the same job.
- demand: 2 objection(s). The table ATTACH fix (0001) costs little: an AccessShareLock on ancestors, the same lock a cold cache already takes. Its demand gap only supports a not-now outcome.

### Precedents retrieved

- P1: [Revert lowering of lock level for ATTACH PARTITION](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8ca984b2c739e096c08f260f477792495f4dfe4) (revert, Reverted, 2018-04-12) - judged not relevant
- P2: [Revert "Don't lock partitions pruned by initial pruning"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1722d5eb05d8e5d2e064cd1798abcae4f296ca9d) (revert, Reverted, 2025-05-22) - judged not relevant
- P3: [Delay locking partitions during query execution](https://commitfest.postgresql.org/patch/1897/) (cf, Withdrawn, 2019-03-05) - judged not relevant; reason summarised from its thread
- P4: [Revert "Move PartitionPruneInfo out of plan nodes into PlannedStmt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5472743d9e8583638a897b47558066167cc14583) (revert, Reverted, 2023-05-04) - judged not relevant
- P5: [Tab completion for ATTACH PARTITION](https://commitfest.postgresql.org/patch/4561/) (cf, Returned with feedback, 2024-01-29) - judged not relevant; reason summarised from its thread
- P6: [ALTER TABLE SET LOGGED/UNLOGGED on a partitioned table does nothing silently](https://commitfest.postgresql.org/patch/2857/) (cf, Withdrawn, 2021-04-07) - used in the review; reason summarised from its thread
- P7: [Revert "Make pg_dump emit ATTACH PARTITION instead of PARTITION OF"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a1ec7402e9a0392f95bc79542af0efcd5c6e7929) (revert, Reverted, 2019-05-06) - judged not relevant
- P8: [Autovacuum (analyze) on partitioned tables for ATTACH/DETACH/DROP commands](https://commitfest.postgresql.org/patch/3188/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P9: [Prevent ALTER TABLE DROP NOT NULL on child tables if parent column has it](https://commitfest.postgresql.org/patch/646/) (cf, Rejected, 2016-06-16) - judged not relevant; reason summarised from its thread
- P10: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P11: [Reduce lock level for ALTER TABLE ... ADD CHECK](https://commitfest.postgresql.org/patch/3088/) (cf, Withdrawn, 2021-11-21) - judged not relevant; reason summarised from its thread
- P12: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant; reason summarised from its thread
- P13: [Allow ordered partition scans in more cases](https://commitfest.postgresql.org/patch/4198/) (cf, Withdrawn, 2023-06-24) - judged not relevant
- P14: [Lazy JIT IR code generation to increase JIT speed with partitions](https://commitfest.postgresql.org/patch/3071/) (cf, Withdrawn, 2023-01-27) - judged not relevant
- P15: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P16: [Revert analyze support for partitioned tables](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8127b7390119c21479f5ce495b7d2168930e82) (revert, Reverted, 2021-08-16) - judged not relevant
- P17: [alter table set TABLE ACCESS METHOD](https://commitfest.postgresql.org/patch/3020/) (cf, Withdrawn, 2021-06-08) - judged not relevant
- P18: [quick and dirty fix: Quadratic planning time for ordered paths over partitioned tables](https://commitfest.postgresql.org/patch/5516/) (cf, Rejected, 2025-02-03) - judged not relevant
- P19: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant
- P20: [Make the optimiser aware of partitions ordering](https://commitfest.postgresql.org/patch/1093/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P21: [Revert "Teach DSM registry to ERROR if attaching to an uninitialized entry."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2dd506b859cfacf8afaae81256ddec8cca0c0d6f) (revert, Reverted, 2025-11-26) - judged not relevant
- P22: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P23: [Restructure ALTER TABLE notes to clarify table rewrites and verification scans](https://commitfest.postgresql.org/patch/3604/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P24: [extended stats on partitioned tables](https://commitfest.postgresql.org/patch/3332/) (cf, Withdrawn, 2021-10-10) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 174 s; cost reported by the CLI: $2.55
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 91 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:51:32.153Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
