# Design pre-review: extended statistic (examine_variable) ignored CollateExpr

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6397](https://commitfest.postgresql.org/patch/6397/); [thread](https://www.postgresql.org/message-id/flat/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is sound: the planner should use expression statistics that match a COLLATE expression. The design is not settled. standards-compat-2 shows the v6 retry may match statistics built under a different collation, and maintenance-fit-3 shows the thread states no single matching rule. Planning cost is unmeasured for v6 after the cost objection in message #3 (maintenance-fit-1), and the stable target is not argued (standards-compat-1).

Not checked: Without the code, this review cannot confirm whether the v6 retry compares collations, how much the second pass and the walker cost, or whether new helper signatures affect a back-patch.

## Prior attempts

None found before 2026-01-13.

## Objections a reviewer is likely to raise

### 1. Fallback match may reuse statistics built under another collation

A user creates statistics on (a COLLATE case_insensitive) and runs GROUP BY a COLLATE "C". The exact match fails, and the retry strips the top RelabelType on both sides, so the two expressions match. The planner then uses the case-insensitive ndistinct of 32 for a grouping that returns 52 groups, which is the bug from message #1 in reverse.

Message #1 argues that ndistinct under one collation does not hold under another. Messages #7 and #9 keep the old strip-and-compare as a fallback for compatibility, and from v6 it also runs when only the statistics expression has a COLLATE. Nothing in the thread says whether the retry checks that the stripped collations agree, or limits itself to deterministic collations, where the statistics would be the same.

**Question:** When the retry strips a COLLATE from either side, does it check that the collations used by the query and the statistics agree, or are both deterministic?

<sub>fixable, level 2, planner behaviour change; messages [#1](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/tencent_974EEA9CC644F88FB6235A8E89F3EF064B07%40qq.com), [#8](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com); `standards-compat-2`</sub>

### 2. Planning cost of v6 two-pass lookup is not measured

A user plans many queries on varchar columns, whose Vars carry an implicit top-level RelabelType, against tables with several expression indexes and statistics objects. v6 compares every index and statistics expression twice when no exact match exists, and also runs a new walker inside strip_all_phvs_mutator. Planning time rises for users who never wrote COLLATE.

examine_variable runs for nearly every clause and grouping expression the planner estimates. In message #3, Tom Lane objected to the cost of v2. The only measurement, in message #8, covers v5 on varchar columns with no index or statistics expressions present. v6 changed the retry conditions and was not measured.

**Question:** What does planning time look like with v6 for varchar and text columns on tables with many expression indexes and statistics objects, with and without PlaceHolderVars?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/453227.1786895971%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com); `maintenance-fit-1`</sub>

### 3. Back-patching a planner estimate change to stable branches

A site with expression statistics or expression indexes on COLLATE expressions installs a minor release. Row estimates for those queries change, for example 52 to 32 or 100 to 26, and some plans flip. The site finds out from slower queries after the update, with no setting to turn the change off.

The CommitFest entry targets stable, but every version since message #6 changes which statistics a COLLATE query matches. Messages #6 and #8 show that each version so far moved estimates for tables the patch did not aim at. Minor releases normally avoid changes to plans unless a wrong result or a crash forces one. The thread does not say which branches have the bug or why a back-patch is worth the plan churn.

**Question:** Which stable branches give the wrong estimate, and why should a plan-changing estimate fix go there instead of master only?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAGRkXqTZjRj2x0vKFbT6pM0%2B-_1kvKbmSPA9HwCX_wB-BkmynA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CAGRkXqQQZPSPd6MC8iUBoBACLUGe2TAUw7%2BBO2yRt_J7YcfnmA%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com); `standards-compat-1`</sub>

### 4. Ad hoc matching rules in examine_variable without an engaged owner

A later patch changes how expressions in index or statistics definitions are normalized, for example how CollateExpr is stripped. That patch has to keep the exact pass, the retry pass and the per-side RelabelType flags consistent. If it misses one, estimates change without any test failing, and users report it releases later.

v6 adds three flags that decide which comparisons the retry runs. The retry keeps the old behaviour from before 7e9f852a as a compatibility fallback, and the thread does not state the rule it implements. Each of v4, v5 and v6 fixed a case the previous version broke, which shows the rules are easy to get wrong. The original author has been silent since message #1, Richard Guo was CC'd in message #4 and did not reply, and no committer has reviewed the rewrite.

**Question:** Can the rule for when an expression matches statistics be stated once, so that index and statistics normalization and examine_variable share one helper and do not diverge?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CACJufxGLCiyhM%2BP0gxesg2x--PTrMY3PszqSqOq_H4QS_oq3Jg%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/tencent_212DD484ECF7882505B551630CFC74DB0C06%40qq.com), [#7](https://www.postgresql.org/message-id/tencent_974EEA9CC644F88FB6235A8E89F3EF064B07%40qq.com), [#9](https://www.postgresql.org/message-id/tencent_1CE0A7DAF468F52662BF752CA1A8771A3909%40qq.com); `maintenance-fit-3`</sub>

## Questions for the author

1. When the v6 retry strips a top RelabelType from either side, does it check that the query collation and the statistics collation agree, or that both are deterministic?
2. Does v6 still change the message #1 demo estimate from 52 to 32 rows?
3. What is the planning time with v6 for varchar and text columns on tables with many expression indexes and statistics objects, with and without PlaceHolderVars?
4. Which stable branches give the wrong estimate, and why should a plan-changing estimate fix go there instead of master only?
5. Can the rule for when a query expression matches an index or statistics expression be stated once and implemented in one shared helper?

## Touch points

- **planner behaviour change:** The patch changes which statistics examine_variable matches for COLLATE expressions, so row estimates and plans change.
- **backpatchability:** The CommitFest entry targets stable, and the change alters row estimates in minor releases.
- **maintenance burden ("who maintains this in 10 years"):** v6 adds an exact pass, a retry pass and per-side RelabelType flags that later normalization changes must keep consistent.
- **performance on the hot path:** examine_variable runs for most estimated clauses, and v6 adds a second comparison pass and a walker in strip_all_phvs_mutator.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows selfuncs.c and extended statistics matching. Tom Lane objected once to the cost of v2 in message #3, and no committer has commented on v3 to v6.
- Priority: fix design first; audience niche; review cost medium. The fix helps users who combine COLLATE expressions with expression statistics or indexes, and it changes a planner path that every query uses. The patch is small, but v4, v5 and v6 each fixed a regression from the previous version, and no code was read for this review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
