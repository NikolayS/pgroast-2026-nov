# Design pre-review: Convert ALL SubLinks to ANY SubLinks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6627](https://commitfest.postgresql.org/patch/6627/); [thread](https://www.postgresql.org/message-id/flat/CAMbWs48P9d8p_MftXr7ZPb86HvOLu%3D3ovNz5By85JvSu9epNQA%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the rewrite reuses the existing ANY sublink code in one planner file. Three fixable points remain. standards-compat-1 is a reproducible error in v2 for a valid operator class (message #8). standards-compat-2 is the unmeasured risk of worse plans after pull-up. already-exists-1 notes that NOT IN already gives the hashed SubPlan for `<> ALL`, and the thread has no benchmark.

Not checked: Without the code, this review could not check that the rewrite keeps NULL semantics in every placement. It also could not check which operator metadata the rewrite rederives after negation.

## Prior attempts

- [Convert NOT IN to anti-joins when possible](https://www.postgresql.org/message-id/flat/CAKJS1f82pqjqe3WT9_xREmXyG20aOkHc-XqkKZG_yMA7JVJ3Tw@mail.gmail.com) (related; Returned with feedback, 2019-11-28). "It has been close to three months since this last update, so marked the patch as returned with feedback." ([message](https://www.postgresql.org/message-id/20191128023021.GG237562%40paquier.xyz)) Different now: P7 converted NOT IN to an anti-join with its own not-null proofs. This patch only rewrites ALL to a negated ANY and leaves pull-up to the existing and pending ANY rules.
- [NOT IN to ANTI JOIN transformation](https://www.postgresql.org/message-id/flat/1550706289606-0.post@n3.nabble.com) (related; Returned with feedback, 2020-04-08). The visible messages do not show why. The last shown message is a rebased patch from June 2019, and no later reply appears before the April 2020 decision. Earlier, David Rowley noted that the added OR ... IS NULL join clauses force a nested loop join, and moved his competing patch to a separate thread. (AI summary of the thread) Different now: P4 added OR ... IS NULL join clauses to handle nullable columns. This patch adds no NULL handling of its own and depends on the ANY and NOT ANY pull-up rules.
- [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (related; Rejected, 2018-11-23). "So I'm inclined to reject this patch as probably being a net loss in almost all cases. If you can show any real-world cases where it wins, we could reconsider." ([message](https://www.postgresql.org/message-id/29888.1542319596%40sss.pgh.pa.us)) Different now: P2 removed hand-written NOT NOT, which is rare in real queries. `<> ALL` is standard syntax, and the rewrite runs only when an ALL sublink is present, so the net-loss reason applies less.

## Objections a reviewer is likely to raise

### 1. The main case, <> ALL, already has a hashed path via NOT IN

A user who writes `x <> ALL (select ...)` can write `x NOT IN (select ...)` today with identical semantics, NULLs included, and already gets a hashed SubPlan. The patch adds a permanent planner rewrite whose main gain is a spelling that users can already change. Users of the other operators, such as `< ALL`, get help, but the proposal does not show that such users exist.

`NOT IN` is parsed as `NOT (x = ANY (...))`, which is the same tree this patch builds for `<> ALL`, so hashing for that form exists already. Pull-up into anti-joins depends on the separate NOT ANY patch noted in message #1, and that patch would cover NOT IN as well. The motivation in message #1 comes from a paper, and no field report of slow `op ALL` queries appears in the thread. [P2] rejected a pull_up_sublinks rewrite of hand-written forms because no real-world wins were shown.

Precedent: [Pull up sublink of type 'NOT NOT (expr)'](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com) (Rejected, 2018-11-23)

**Question:** Which real queries, for example from ORMs or query generators, use `op ALL` forms that users cannot rewrite as NOT IN, and how much faster do they run?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAMbWs48P9d8p_MftXr7ZPb86HvOLu%3D3ovNz5By85JvSu9epNQA%40mail.gmail.com); `already-exists-1`</sub>

### 2. Rewrite turns valid row-comparison ALL queries into errors

A user has a btree operator class that passes amvalidate() but whose operator's NEGATOR is outside the family. After upgrade, a row-wise `op ALL` query that returned rows on master fails with "operator ... is not a member of opfamily". The user finds out when the query runs in production.

Message #8 shows the rewrite swaps in the negators but keeps the original rcexpr->opfamilies, so the result can be an inconsistent RowCompareExpr. The rewrite assumes a contract, that a NEGATOR is always a member of the same family, which Postgres does not enforce for user-defined operators. Rui posted three possible fixes, and the author has not yet chosen one. The brief also says non-btree operators and commutated row comparisons were not checked.

**Question:** Which conditions on negator and opfamily membership does the rewrite require, and does the patch check each of them before rewriting?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/CAHWVJhHumJO_s7R8Uy2s0Q_8XwQMqW7dPaawP_znqGNH-ryhxQ%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Existing ALL queries change plan shape with no opt-out

An application written with `<> ALL` runs a tuned plan today. After upgrade, pull-up merges the subquery into the outer join problem, and the join count goes past join_collapse_limit or picks up LATERAL limits. The query slows down, and the only remedies are raising limits or rewriting generated SQL.

Message #2 and message #5 report that new sublink pull-ups often cause user complaints for this reason. Message #3 answers that the join search is at fault and users should tune join_collapse_limit, but that is a workaround for users, not a design answer. The proposal shows no benchmark and offers no per-query or per-session way to keep today's ALL SubPlan behaviour.

**Question:** What can a user do, short of rewriting the query or raising join_collapse_limit, when the pulled-up ALL sublink gives a worse plan than before?

<sub>fixable, level 2, planner behaviour change; messages [#2](https://www.postgresql.org/message-id/0ea8fe57-05c5-428e-936c-fe1dfaed2cc5%40gmail.com), [#3](https://www.postgresql.org/message-id/CAMbWs48gsDch407qHLWXeS5TnEUCLsLDsUYLjNQiNOUJZg2iVQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/4c04bc45-04a6-4284-9293-e645851f490a%40gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. Which conditions on negator and opfamily membership does the rewrite require, and does the patch check each one for btree, non-btree and commutated row comparisons?
2. Which `op ALL` queries run faster with v2, and by how much on a benchmark?
3. Which real queries, for example from ORMs or query generators, use `op ALL` forms that users cannot write as NOT IN?
4. What can a user do when a pulled-up ALL sublink gives a worse plan, other than rewrite the query or raise join_collapse_limit?
5. What does this patch gain alone for `op ALL`, without the pending NOT ANY pull-up patch that message #1 mentions?

## Touch points

- **ABI / extension API:** The rewrite assumes an operator's negator is in the same opfamily, which user-defined operator classes need not satisfy (message #8).
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Rewrite turns valid row-comparison ALL queries into errors
- **planner behaviour change:** Existing ALL sublinks can become hashed ANY SubPlans, semijoins or anti-joins.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a permanent rewrite in prepjointree.c that must keep operator metadata consistent after negation.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** NOT IN already parses to the tree this patch builds for `<> ALL` and already gets a hashed SubPlan.

## Who would commit it, and when

- Owner: qualified committer engaged. The maintenance-fit note says the author, Richard Guo, is a planner committer. A second person who knows sublink pull-up and row-comparison opfamilies has not reviewed it in the visible thread.
- Priority: proceed; audience unclear; review cost medium. The change is confined to prepjointree.c and reuses existing ANY sublink code, so the cost after commit is small. The thread shows no user report or benchmark, and the opfamily error in message #8 needs an answer first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
