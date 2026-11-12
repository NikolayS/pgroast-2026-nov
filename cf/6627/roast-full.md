# Design pre-review: Convert ALL SubLinks to ANY SubLinks

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6627](https://commitfest.postgresql.org/patch/6627/) (Needs review); authors: Richard Guo (richardguo)
- Thread: [Convert ALL SubLinks to ANY SubLinks](https://www.postgresql.org/message-id/flat/CAMbWs48P9d8p_MftXr7ZPb86HvOLu%3D3ovNz5By85JvSu9epNQA%40mail.gmail.com), started 2026-02-26
- Messages read: 8 of 8
- Patch prose read: v2-0001-Convert-ALL-SubLinks-to-ANY-SubLinks.patch, v1-0001-Convert-ALL-SubLinks-to-ANY-SubLinks.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the rewrite reuses the existing ANY sublink code in one planner file. Three fixable points remain. standards-compat-1 is a reproducible error in v2 for a valid operator class (message #8). standards-compat-2 is the unmeasured risk of worse plans after pull-up. already-exists-1 notes that NOT IN already gives the hashed SubPlan for `<> ALL`, and the thread has no benchmark.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the rewrite keeps NULL semantics in every placement. It also could not check which operator metadata the rewrite rederives after negation.

## The proposal, restated

**Problem.** The planner runs ALL sublinks as a nested-loop SubPlan, and they get none of the ANY sublink optimizations: hashed SubPlans and pull-up into semijoins or anti-joins. Queries using `<> ALL` (a common way to write NOT IN semantics) or other `op ALL` forms pay for this.

**Proposed design.** In prepjointree.c the planner replaces each operator with its negator and applies De Morgan's laws: `foo op ALL (sub)` becomes `NOT (foo negator_op ANY (sub))`, and `NOT (foo op ALL (sub))` becomes `foo negator_op ANY (sub)`. The rewritten sublink then uses hashing or pull-up when it qualifies. If an operator has no negator, the ALL sublink stays unchanged.

**User-visible surface.**
- Plan shapes change: ALL sublinks can appear as hashed ANY SubPlans, semijoins or anti-joins in EXPLAIN output.

**What the author claims.**
- Converted sublinks can use hashed SubPlans and can be flattened into semijoins or anti-joins.
- Worst case is a nested-loop ANY SubPlan with the same performance as today, so there are no regressions.
- The negated ANY subplan stops on the same inner tuple as the ALL subplan; the only cost added is one NOT per outer tuple.
- Join-order problems after pull-up belong to the join search and are handled with join_collapse_limit and geqo_threshold.

**Thread state.** Lepikhov disputes the no-regression claim, citing larger join problems and LATERAL limits after pull-up; solai v tested v2 and gave +1. Rui Zhao found that v2 errors for a valid custom operator class whose negator is outside the opfamily, and posted a fix that Richard has not yet answered.

## 2. Prior attempts

### Convert NOT IN to anti-joins when possible (related)

- **Link:** [https://commitfest.postgresql.org/patch/2020/](https://commitfest.postgresql.org/patch/2020/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKJS1f82pqjqe3WT9_xREmXyG20aOkHc-XqkKZG_yMA7JVJ3Tw@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKJS1f82pqjqe3WT9_xREmXyG20aOkHc-XqkKZG_yMA7JVJ3Tw@mail.gmail.com)
- **Date:** decided 2019-11-28; started 2019-03-05
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Antonin Houska (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Antonin Houska showed that join quals used as not-null proofs can be wrong when the NOT IN is in a JOIN/ON clause, because join order may let NULLs reach the anti-join. David Rowley limited the optimisation to WHERE-clause quals, and Antonin then sent review comments on that version. The author said he had no time to respond, and the entry was returned with feedback after about three months with no update.
- **Reason (quote):** "It has been close to three months since this last update, so marked the patch as returned with feedback." ([message](https://www.postgresql.org/message-id/20191128023021.GG237562%40paquier.xyz))
- **What is different this time:** P7 converted NOT IN to an anti-join with its own not-null proofs. This patch only rewrites ALL to a negated ANY and leaves pull-up to the existing and pending ANY rules.

### NOT IN to ANTI JOIN transformation (related)

- **Link:** [https://commitfest.postgresql.org/patch/2023/](https://commitfest.postgresql.org/patch/2023/)
- **Thread:** [https://www.postgresql.org/message-id/flat/1550706289606-0.post@n3.nabble.com](https://www.postgresql.org/message-id/flat/1550706289606-0.post@n3.nabble.com), [https://www.postgresql.org/message-id/flat/61BFFE8C-F884-4936-BB28-0098283F4719@amazon.com](https://www.postgresql.org/message-id/flat/61BFFE8C-F884-4936-BB28-0098283F4719@amazon.com)
- **Date:** decided 2020-04-08; started 2019-02-20
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The visible messages do not show why. The last shown message is a rebased patch from June 2019, and no later reply appears before the April 2020 decision. Earlier, David Rowley noted that the added OR ... IS NULL join clauses force a nested loop join, and moved his competing patch to a separate thread.
- **Reason (quote):** quote unavailable
- **What is different this time:** P4 added OR ... IS NULL join clauses to handle nullable columns. This patch adds no NULL handling of its own and depends on the ANY and NOT ANY pull-up rules.

### Pull up sublink of type 'NOT NOT (expr)' (related)

- **Link:** [https://commitfest.postgresql.org/patch/1831/](https://commitfest.postgresql.org/patch/1831/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)
- **Date:** decided 2018-11-23; started 2018-10-17
- **Outcome:** Rejected (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane argued that pull_up_sublinks runs before transformations that could create NOT NOT, so it only sees NOT NOT that a user wrote by hand, and no one showed that this happens in real queries. Without such evidence the extra code and planning cycles would be a net loss for everyone else. Running expression simplification before pullup would need a second simplification pass, which he judged not worth the cost. The author agreed and asked for the patch to be rejected.
- **Reason (quote):** "So I'm inclined to reject this patch as probably being a net loss in almost all cases. If you can show any real-world cases where it wins, we could reconsider." ([message](https://www.postgresql.org/message-id/29888.1542319596%40sss.pgh.pa.us))
- **What is different this time:** P2 removed hand-written NOT NOT, which is rare in real queries. `<> ALL` is standard syntax, and the rewrite runs only when an ALL sublink is present, so the net-loss reason applies less.


## 3. Objections a reviewer is likely to raise

### 1. The main case, <> ALL, already has a hashed path via NOT IN

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), planner behaviour change · `already-exists-1`

A user who writes `x <> ALL (select ...)` can write `x NOT IN (select ...)` today with identical semantics, NULLs included, and already gets a hashed SubPlan. The patch adds a permanent planner rewrite whose main gain is a spelling that users can already change. Users of the other operators, such as `< ALL`, get help, but the proposal does not show that such users exist.

`NOT IN` is parsed as `NOT (x = ANY (...))`, which is the same tree this patch builds for `<> ALL`, so hashing for that form exists already. Pull-up into anti-joins depends on the separate NOT ANY patch noted in message #1, and that patch would cover NOT IN as well. The motivation in message #1 comes from a paper, and no field report of slow `op ALL` queries appears in the thread. [P2] rejected a pull_up_sublinks rewrite of hand-written forms because no real-world wins were shown.

**Precedents.**
- [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (Rejected, 2018-11-23); thread: [link](https://www.postgresql.org/message-id/flat/CAN_9JTxpgxnTa_EPvmfuy6vogY0smQmYbEvbJhx=bD1iAjK5BQ@mail.gmail.com)

Thread: [#1 (Richard Guo, 2026-02-26)](https://www.postgresql.org/message-id/CAMbWs48P9d8p_MftXr7ZPb86HvOLu%3D3ovNz5By85JvSu9epNQA%40mail.gmail.com)

**Question:** Which real queries, for example from ORMs or query generators, use `op ALL` forms that users cannot rewrite as NOT IN, and how much faster do they run?

Counter-check: held. NOT IN does parse to the tree this patch builds for `<> ALL`, and the thread shows no user report or benchmark for other `op ALL` forms. [P2] is weak support, because `<> ALL` is a standard spelling, unlike hand-written NOT NOT.

### 2. Rewrite turns valid row-comparison ALL queries into errors

fixable · level 2, rubric item 7 · design, planner behaviour change, SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user has a btree operator class that passes amvalidate() but whose operator's NEGATOR is outside the family. After upgrade, a row-wise `op ALL` query that returned rows on master fails with "operator ... is not a member of opfamily". The user finds out when the query runs in production.

Message #8 shows the rewrite swaps in the negators but keeps the original rcexpr->opfamilies, so the result can be an inconsistent RowCompareExpr. The rewrite assumes a contract, that a NEGATOR is always a member of the same family, which Postgres does not enforce for user-defined operators. Rui posted three possible fixes, and the author has not yet chosen one. The brief also says non-btree operators and commutated row comparisons were not checked.

Thread: [#8 (Rui Zhao, 2026-11-09)](https://www.postgresql.org/message-id/CAHWVJhHumJO_s7R8Uy2s0Q_8XwQMqW7dPaawP_znqGNH-ryhxQ%40mail.gmail.com)

**Question:** Which conditions on negator and opfamily membership does the rewrite require, and does the patch check each of them before rewriting?

Counter-check: held. Message #8 gives a reproducible case where v2 turns a working query into an error. A fix is posted, but the author has not adopted one, and non-btree and commutated cases are unchecked.

### 3. Existing ALL queries change plan shape with no opt-out

fixable · level 2, rubric item 8 · planner behaviour change · `standards-compat-2`

An application written with `<> ALL` runs a tuned plan today. After upgrade, pull-up merges the subquery into the outer join problem, and the join count goes past join_collapse_limit or picks up LATERAL limits. The query slows down, and the only remedies are raising limits or rewriting generated SQL.

Message #2 and message #5 report that new sublink pull-ups often cause user complaints for this reason. Message #3 answers that the join search is at fault and users should tune join_collapse_limit, but that is a workaround for users, not a design answer. The proposal shows no benchmark and offers no per-query or per-session way to keep today's ALL SubPlan behaviour.

Thread: [#2 (Andrei Lepikhov, 2026-02-26)](https://www.postgresql.org/message-id/0ea8fe57-05c5-428e-936c-fe1dfaed2cc5%40gmail.com), [#3 (Richard Guo, 2026-02-27)](https://www.postgresql.org/message-id/CAMbWs48gsDch407qHLWXeS5TnEUCLsLDsUYLjNQiNOUJZg2iVQ%40mail.gmail.com), [#5 (Andrei Lepikhov, 2026-02-27)](https://www.postgresql.org/message-id/4c04bc45-04a6-4284-9293-e645851f490a%40gmail.com)

**Question:** What can a user do, short of rewriting the query or raising join_collapse_limit, when the pulled-up ALL sublink gives a worse plan than before?

Counter-check: held. Lepikhov's concern in messages #2 and #5 is concrete, and the thread answers it only with advice to tune settings. The lack of an opt-out matches existing ANY pull-up, so the open point is the unmeasured regression risk for existing ALL queries.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The rewrite assumes an operator's negator is in the same opfamily, which user-defined operator classes need not satisfy (message #8). |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Rewrite turns valid row-comparison ALL queries into errors |
| new GUC | no |  |
| planner behaviour change | **yes** | Existing ALL sublinks can become hashed ANY SubPlans, semijoins or anti-joins. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a permanent rewrite in prepjointree.c that must keep operator metadata consistent after negation. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | NOT IN already parses to the tree this patch builds for `<> ALL` and already gets a hashed SubPlan. |

## 5. Questions the author must answer before anyone reviews code

1. Which conditions on negator and opfamily membership does the rewrite require, and does the patch check each one for btree, non-btree and commutated row comparisons?
2. Which `op ALL` queries run faster with v2, and by how much on a benchmark?
3. Which real queries, for example from ORMs or query generators, use `op ALL` forms that users cannot write as NOT IN?
4. What can a user do when a pulled-up ALL sublink gives a worse plan, other than rewrite the query or raise join_collapse_limit?
5. What does this patch gain alone for `op ALL`, without the pending NOT ANY pull-up patch that message #1 mentions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The maintenance-fit note says the author, Richard Guo, is a planner committer. A second person who knows sublink pull-up and row-comparison opfamilies has not reviewed it in the visible thread.
- **Liability:** The owner signs for plan changes in every existing query with an ALL sublink, and for correct negation with user-defined operators, with no end date.
- **Author follow-up:** The author posted v2 in message #6 and kept the entry open across three CommitFests. Message #8 has no reply yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The change is confined to prepjointree.c and reuses existing ANY sublink code, so the cost after commit is small. The thread shows no user report or benchmark, and the opfamily error in message #8 needs an answer first.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Anti-join pull-up of rewritten <> ALL needs a shown NULL proof. Refuted (no failure scenario): The rewrite of `<> ALL` builds the same tree the parser builds for NOT IN, so NULL handling after pull-up belongs to the NOT ANY pull-up code. The proposal shows no wrong result, and [P7] concerned a different patch's not-null proofs.
- `maintenance-fit-1` Negation keeps operator metadata that the new operator may not match. Refuted (not specific): Duplicate of standards-compat-1. The added point about fields that future patches might miss names no concrete field or case.
- `maintenance-fit-2` No way to undo the pull-up when a plan regresses. Refuted (not specific): Duplicate of standards-compat-2.
- `demand-1` No field report or measurement shows slow ALL sublinks matter. Refuted (not specific): Duplicate of already-exists-1, which makes the same demand point with the more concrete NOT IN equivalence.
- `demand-2` Existing ALL queries change plans without asking, and have no opt-out. Refuted (not specific): Duplicate of standards-compat-2. The item 4 framing also fits poorly, because the users who may pay are the same users of ALL sublinks who gain.

### Review angles

- prior-attempts: 1 objection(s). [P1] was skipped. Its LATERAL and code-duplication problems came from pulling up correlated ANY sublinks, and this patch reuses the existing ANY pull-up rules unchanged.
- standards-compat: 2 objection(s). The De Morgan rewrite itself keeps SQL three-valued semantics, and negate_clause() already uses negators to flip ANY and ALL for array expressions, so trusting negators follows existing practice.
- replication-ondisk: 0 objection(s). The patch changes only planner prep code in prepjointree.c and regression tests, so it adds no on-disk, WAL, catalog, wire-protocol or ABI state. Stored views and rules keep the original ALL sublink in their parse trees because the rewrite happens at plan time, so pg_dump and pg_upgrade are unaffected.
- security: 0 objection(s). The proposal has no privilege or exposure aspect that is new. It rewrites an ALL sublink into a negated ANY sublink by swapping each operator for its declared negator. The planner already trusts NEGATOR the same way when it simplifies NOT (a < b) into a >= b. EXECUTE on the negator's function is still checked at run time, so a missing grant gives an error and no escalation. After pull-up, the existing leakproof and security barrier rules for quals still apply. A user-defined operator whose NEGATOR does not match its meaning gives wrong results only to queries that use that operator, as it does today. The opfamily error in message #8 is a correctness problem, outside this angle.
- maintenance-fit: 2 objection(s). Fit is good otherwise. The author is a planner committer, the change is confined to prepjointree.c, and it reuses existing ANY sublink machinery. The cost for queries without ALL sublinks appears negligible, unlike the planning-cycle concern in [P2].
- already-exists: 1 objection(s). No extension or hook covers this rewrite. A planner rewrite of this kind belongs in core, so the only objection from this angle concerns demand beyond the NOT IN spelling.
- demand: 2 objection(s). The demand case is thin, not absent, so both objections are "not now, show demand". A few real cases with numbers would settle them.

### Precedents retrieved

- P1: [Pulling up direct-correlated ANY_SUBLINK](https://commitfest.postgresql.org/patch/3941/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [Pull up sublink of type 'NOT NOT (expr)'](https://commitfest.postgresql.org/patch/1831/) (cf, Rejected, 2018-11-23) - used in the review; reason summarised from its thread
- P3: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P4: [NOT IN to ANTI JOIN transformation](https://commitfest.postgresql.org/patch/2023/) (cf, Returned with feedback, 2020-04-08) - used in the review; reason summarised from its thread
- P5: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P6: [Memoize ANTI and SEMI JOIN inner](https://commitfest.postgresql.org/patch/5636/) (cf, Withdrawn, 2025-06-30) - judged not relevant; reason summarised from its thread
- P7: [Convert NOT IN to anti-joins when possible](https://commitfest.postgresql.org/patch/2020/) (cf, Returned with feedback, 2019-11-28) - used in the review; reason summarised from its thread
- P8: [Protect syscache from bloating with negative cache entries](https://commitfest.postgresql.org/patch/931/) (cf, Returned with feedback, 2021-10-05) - judged not relevant; reason summarised from its thread
- P9: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant
- P10: [Improve Append/MergeAppend EXPLAIN output when all subplans are pruned.](https://commitfest.postgresql.org/patch/2102/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P11: [Negative RelfilenumberMap cache entries from pg_filenode_relation()](https://commitfest.postgresql.org/patch/6066/) (cf, Withdrawn, 2025-09-18) - judged not relevant
- P12: [Correct negative/zero year in to_date/to_timestamp](https://commitfest.postgresql.org/patch/533/) (cf, Rejected, 2016-03-11) - judged not relevant
- P13: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P14: [INNER JOIN removals](https://commitfest.postgresql.org/patch/27/) (cf, Rejected, 2015-02-14) - judged not relevant
- P15: [Revert "Tweak "line" test to avoid negative zeros on some platforms"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1f7a47912af2206698d14650f2149fa951b8ce07) (revert, Reverted, 2013-10-25) - judged not relevant
- P16: [Revert "Skip redundant anti-wraparound vacuums"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dd9ac7d5d80608a640bb82cffb6a805ce84cf112) (revert, Reverted, 2020-03-30) - judged not relevant
- P17: [A little cosmetic to convert_VALUES_to_ANY()](https://commitfest.postgresql.org/patch/5937/) (cf, Withdrawn, 2025-08-04) - judged not relevant
- P18: [Add a feature that enables row-wise comparison partition pruning](https://commitfest.postgresql.org/patch/2654/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant
- P20: [revert: Transform OR clauses to ANY expression](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ff9f72c68f678ded340b431c3e280fe56644a3e7) (revert, Reverted, 2024-04-09) - judged not relevant
- P21: [Discarding DISCARD ALL](https://commitfest.postgresql.org/patch/2919/) (cf, Withdrawn, 2021-06-21) - judged not relevant
- P22: [Increasing the default values of from_collapse_limit, join_collapse_limit, and/or geqo_threshold](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P23: [Subplan result cache](https://commitfest.postgresql.org/patch/1646/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P24: [parse-time support function for "any" datatype](https://commitfest.postgresql.org/patch/2081/) (cf, Withdrawn, 2021-03-09) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 152 s; cost reported by the CLI: $1.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 89 text field(s) checked, 2 finding(s), 2 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:56:49.298Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
