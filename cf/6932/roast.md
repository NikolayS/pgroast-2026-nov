# Design pre-review: Improve row estimation with multi-column unique indexes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6932](https://commitfest.postgresql.org/patch/6932/); [thread](https://www.postgresql.org/message-id/flat/CAO6_XqoiwRoVFS1x1oaUpwUi5kKwOj1S1JVYBO2QX91ORJJoxg%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is accepted by the only reviewer, and no objection survived at level 1. Two level 2 objections survived. standards-compat-1: v3 gives an index path rows=1 while the relation estimate stays high, and message #3 does not answer the README rule raised in message #2. standards-compat-3: v2 and v3 disagree on deferrable unique constraints. The thread needs to settle both before code review.

Not checked: The code was not read. Not judged: whether is_path_unique handles IS NULL, arrays and partial-index predicates correctly, and which callers read the path row count that v3 changes.

## Prior attempts

- [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (related; Reverted, 2016-06-07). "We found multiple bugs in this feature, and there was concern about possible planner slowdown (though to be fair, exhibiting a very large slowdown proved difficult)." Different now: P24 used foreign keys to infer join selectivity across relations. This proposal uses unique indexes to bound one relation's scan to one row, with a much smaller scope.

## Objections a reviewer is likely to raise

### 1. Paths of one relation get different row counts for the same clauses

A query joins visitor on (org, name). The index scan path reports rows=1, while the seq scan, bitmap path and the join size, which use the relation estimate, still report about 2149. EXPLAIN shows a scan with rows=1 under a join sized from 2149, and the user cannot tell which estimate the planner acted on.

Message #2 says the optimizer README expects paths for the same relation and parameterization to share one row count. v3 keeps the path-level design and sets rows only on the IndexPath. Message #3 answers with partial unique indexes but says nothing about the README rule or about join estimates. A committer would likely want a relation-level clamp where it is valid, plus a path-level clamp only for partial indexes, with a stated reason for that exception.

**Question:** For which index types and plan shapes may index paths report fewer rows than the relation, and how do join size estimates stay consistent with that?

<sub>fixable, level 2, planner behaviour change; messages [#2](https://www.postgresql.org/message-id/0AF0607F-5FC9-46BD-B593-DE78BEBAAA36%40outlook.com), [#3](https://www.postgresql.org/message-id/CAO6_Xqpg9t6n0iibUwqihfEx3xPwWtRFGenxi%3DtKWoCNqyCcGA%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Deferrable unique constraints changed without a decision in the thread

A transaction loads rows under a deferred unique constraint and then joins against the table on the full key before commit. The planner assumes one row per key and picks a nested loop. Duplicates exist until commit, so the join runs much slower than planned, and nothing in EXPLAIN explains why.

v2 excludes deferrable constraints. Message #3 keeps them and argues that one row is usually closer to the truth. That argument is reasonable, but it extends the B-tree shortcut to multi-column keys without a recorded decision. A committer would likely want the choice made explicitly and documented, because this is the case where the estimate can be wrong by the size of the batch.

**Question:** Should deferrable unique constraints count toward the one-row bound, and what behaviour do bulk loads inside the deferral window get?

<sub>fixable, level 2, planner behaviour change; messages [#2](https://www.postgresql.org/message-id/0AF0607F-5FC9-46BD-B593-DE78BEBAAA36%40outlook.com), [#3](https://www.postgresql.org/message-id/CAO6_Xqpg9t6n0iibUwqihfEx3xPwWtRFGenxi%3DtKWoCNqyCcGA%40mail.gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. How do join size estimates, which use the relation-level row count, stay consistent with an index path that reports rows=1?
2. Why not clamp at relation level where relation_has_unique_index_for() applies, and keep a path-level clamp only for unique partial indexes?
3. Should deferrable unique constraints count toward the one-row bound, given that v2 excludes them and duplicates can exist until commit?
4. Do index AMs other than B-tree get the unique path behaviour, and through which code path?
5. What is the planning-time cost of is_path_unique on queries with many indexes, and do any existing regression plans change?

## Touch points

- **ABI / extension API:** v3 adds a unique_path field to IndexPath in pathnodes.h and touches pathnode.h, which extensions that build index paths see.
- **planner behaviour change:** Row estimates and plan choice change for full-key equality lookups on multi-column unique indexes.
- **maintenance burden ("who maintains this in 10 years"):** A path-level row count that differs from the relation estimate is an exception later planner work must keep in mind.
- **performance on the hot path:** is_path_unique runs each time index paths are built, and the thread shows no planning-time measurement.

## Who would commit it, and when

- Owner: silent. Someone who knows planner size estimation and index costing (costsize.c, selfuncs.c, indxpath.c). No such committer has posted in the three-message thread.
- Priority: fix design first; audience a sizeable minority; review cost medium. Lookups by a composite unique key are common, and the reviewer in message #2 agrees the misestimate is real. v2 and v3 differ on the level of the clamp and on deferrable constraints, and the patch touches core planner costing files.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
