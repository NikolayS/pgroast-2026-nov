# Design pre-review: BUG #19798: "failed to build any 4-way joins" with ROLLUP over a LATERAL subquery that references the nullable s

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7601](https://commitfest.postgresql.org/patch/7601/); [thread](https://www.postgresql.org/message-id/flat/19798-5a5da856f5f71118%40postgresql.org), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real, reproducible and well analysed, and a fix is wanted. The one surviving objection, maintenance-fit-1, is a fixable design point: the patch adds a second rule in join_search_one_level() next to has_join_restriction(), and message #2 does not say whether the three join.sql plan changes of the alternative are worse. A committer who knows the join search code has to choose between the two fixes.

Not checked: Without the code, the exact condition of the exception, its cost when the PlaceHolderVar list is long, and whether it admits extra pairs that change plans on stable branches could not be judged. The prose does not show whether a regression test is included.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. Local pruning exception duplicates the meaning of has_join_restriction()

A later patch changes how PlaceHolderVars or lateral references restrict join order and updates has_join_restriction() only. The bushy-loop exception keeps the old rule. A new "failed to build any N-way joins" error then appears on some grouping-sets or lateral query, found later by a fuzzer or a user.

After this patch, the join search has two places that decide whether a rel takes part in a join-order restriction: has_join_restriction() and the new exception in join_search_one_level(). Nothing fails if they drift apart. Message #2 rejects fixing has_join_restriction() because three join.sql plans change, but it does not say whether those plans get worse. A committer would likely prefer one definition of "has a join restriction", or a fix to the joinrel lateral_referencers that message #1 says is always NULL.

**Question:** Are the three join.sql plan changes from the has_join_restriction() fix actually worse, and if not, why keep a second, local rule?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/19798-5a5da856f5f71118%40postgresql.org), [#2](https://www.postgresql.org/message-id/CAHewXNkpyvue6dx5Q33mHXnTh4uxDHHAw0szbwGCBnyZwHThpA%40mail.gmail.com); `maintenance-fit-1`</sub>

## Questions for the author

1. Are the three join.sql plan changes from the has_join_restriction() fix worse, equal or better than the current plans?
2. Why is a local exception in join_search_one_level() preferable to setting lateral_referencers for joinrels, which message #1 says is always NULL?
3. Which branches should receive the fix, given that the bug reproduces on 14 through master?
4. Does the patch fix the 5-way and 6-way variants from message #1, and does it add regression tests for them?
5. What is the planning-time cost of the new check on queries with many relations and many PlaceHolderVars?

## Touch points

- **planner behaviour change:** The patch changes which bushy-join pairs join_search_one_level() considers in joinrels.c.
- **backpatchability:** The bug reproduces on 14 through master, and the thread does not name the target branches.
- **maintenance burden ("who maintains this in 10 years"):** The new exception is a second join-restriction rule that must stay in step with has_join_restriction().
- **performance on the hot path:** The new check runs in the bushy-join loop of join search, and its planning-time cost is not measured.

## Who would commit it, and when

- Owner: silent. Someone who knows the join search code in joinrels.c and how PlaceHolderVars and lateral references restrict join order. No such committer has posted in the two visible messages.
- Priority: proceed; audience niche; review cost low. The patch is 2.1 KB in one file and fixes an internal planner error on valid SQL that reaches back to 14. The failing query shape is rare and was found by a fuzzer, but the choice between the three fixes needs a reviewer who knows join search.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
