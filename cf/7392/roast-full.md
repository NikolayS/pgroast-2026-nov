# Design pre-review: Let an ordering index scan hand its ORDER BY value to the target list

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7392](https://commitfest.postgresql.org/patch/7392/) (Needs review); authors: Greg Burd (gregburd)
- Thread: [Let an ordering index scan hand its ORDER BY value to the target list](https://www.postgresql.org/message-id/flat/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), started 2026-10-01
- Messages read: 22 of 22
- Patch prose read: v13-0006-Let-an-index-only-scan-win-on-the-index-expressi.patch, v13-0005-Don-t-charge-an-index-only-scan-for-index-expres.patch, v13-0008-Carry-index-computed-values-through-outer-joins-.patch, v13-0001-Compute-the-distance-between-a-point-and-a-box-e.patch, v13-0002-Let-an-ordering-index-scan-hand-its-ORDER-BY-val.patch, v13-0004-Let-an-ordering-index-scan-below-a-join-emit-its.patch, v13-0003-Don-t-charge-an-ordering-index-scan-for-ORDER-BY.patch, v13-0007-Carry-index-computed-values-up-through-joins-and.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support: message #3 agrees the double evaluation should be fixed, and the series answers the main complaints recorded in [P2]. Three design points are still open. maintenance-fit-1: 0007 and 0008 add a path property that nothing checks. standards-compat-1: extension opclasses cannot opt in. prior-attempts-1: nobody working on amgetbatch has said the series fits it. demand-1 and prior-attempts-2 put the cost of 0007-0008 in question. Nothing that survived is fatal.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the agreement between path_target_cost() and the setrefs.c rewrite cannot be checked. The same holds for the outer-join nullingrels matching in 0008 and the EvalPlanQual and reorder-queue value handling in 0002.

## The proposal, restated

**Problem.** When a kNN index scan orders by an operator and the query also selects that expression, the executor evaluates it again per row (a third time on recheck). The planner also charges index scans and index-only scans for expressions they read from the index, so it can pick Seq Scan + Sort. This costs users of expensive distance or ranking functions and expression indexes.

**Proposed design.** A new optional index AM callback says whether an opclass's non-rechecked ORDER BY values equal the operator's result; when it does, setrefs replaces a matching top-level target entry with a reference to the scan's value. Later patches stop charging for such values, give index paths their own PathTarget, and carry the values up through joins, Appends, Gather and outer joins. Patch 0001 separately makes the point-to-box distance exact.

**User-visible surface.**
- New IndexAmRoutine callback amcanreturnorderby(opfamily, opcintype, orderbyop), NULL by default.
- New EXPLAIN VERBOSE line "Order By Values Used: N" on index scans; Output lines gain extra parentheses.
- GiST point_ops and box_ops, SP-GiST quad_point_ops, kd_point_ops and box_ops answer yes.
- <->(point, box), <->(box, point) and ##(point, box) results change in edge cases; SP-GiST box distance now errors on overflow and returns NaN for NaN input.
- Planner plan choices change for queries with costly ORDER BY or index expressions; new add_path_precheck_extras() and add_partial_path_precheck_extras() functions.

**What the author claims.**
- Query output does not depend on plan choice, because a qualifying opclass promises its value equals the operator's result.
- Expression-index kNN queries ran about 88-94% faster; point kNN changes are within noise.
- HammerDB mixes with targeted queries gained up to +196% throughput; TPROC-C and TPROC-H plans and timings are unchanged.
- Carrying values through joins adds up to +49% planning time and more planner memory at eleven rels, only when an expression costs more than 10 times cpu_operator_cost.
- An extension-side workaround using a backend global breaks with nested ordering scans; the core version has no such hazard.

**Thread state.** Matthias van de Meent agreed the double evaluation should be fixed and shaped the per-opclass callback; Oleg Bartunov reported a 1.7-2.2x speedup with a ranked GIN prototype. No one has reviewed v13 or the planner patches 0003-0008, and the entry has no reviewer.

## 2. Prior attempts

### Avoid computing ORDER BY junk columns unnecessarily (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/4717/](https://commitfest.postgresql.org/patch/4717/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2ca5865b-4693-40e5-8f78-f3b45d5378fb@iki.fi](https://www.postgresql.org/message-id/flat/2ca5865b-4693-40e5-8f78-f3b45d5378fb@iki.fi)
- **Date:** decided 2024-02-10; started 2023-12-21
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane, Richard Guo (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane found that the patch changes only the final top-level targetlist, so lower plan nodes still compute the expensive expression and cost estimates do not reflect the saving. He also thought the resjunk enum change was invasive and suggested waiting for the work on unifying index scans and index-only scans. Richard Guo found Assert failures with postgres_fdw and with FOR UPDATE queries, and CFbot later reported test failures with no new version posted.
- **Reason (quote):** "My basic complaint is that this is a band-aid that only touches things at a surface level, whereas I think we need a much deeper set of changes in order to have a plausible solution." ([message](https://www.postgresql.org/message-id/2935085.1703806977%40sss.pgh.pa.us))
- **What is different this time:** P2 removed junk columns only in the final targetlist and changed resjunk to an enum. This series leaves resjunk alone, fixes costing (0003, 0005) and carries values below joins (0004, 0007). The advice to wait for index scan unification is still open.

### Index Only Scan for index expressions (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/738/](https://commitfest.postgresql.org/patch/738/)
- **Thread:** [https://www.postgresql.org/message-id/flat/57B23C4F.7000304@postgrespro.ru](https://www.postgresql.org/message-id/flat/57B23C4F.7000304@postgrespro.ru)
- **Date:** decided 2016-09-28; started 2016-08-15
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Vladimir Sitnikov, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Testing of the sixth patch version by Vladimir Sitnikov found an error ("variable not found in subplan target list") when queries access non-indexed columns alongside an expression index. Robert Haas marked the entry Returned with Feedback because these bugs precluded committing it before the CommitFest ended, while inviting resubmission and saying the effort was good.
- **Reason (quote):** "Based on this report, this patch appears to have bugs that would preclude committing it, so I'm marking it "Returned with Feedback" for this CommitFest, which is due to end shortly." ([message](https://www.postgresql.org/message-id/CA%2BTgmoaiYuarPYv2BWrXG-7XZ1EX75guirx2kTXQY0HxtD%3DWuw%40mail.gmail.com))
- **What is different this time:** P3 added index-only scan matching for expression columns and was returned because of bugs. This series keeps the existing matching and changes costing and path targets (0005, 0006).

### Improve use of expression indexes for ORDER BY (similar idea)

- **Link:** [https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor)
- **Date:** decision date unknown
- **Outcome:** Not worth doing (do_not_want)
- **Who objected:** not recorded in the corpus
- **Reason (quote):** quote unavailable
- **What is different this time:** The row lists a similar goal as not worth doing but records no reason. This series brings measurements and a ranked GIN test from message #16.


## 3. Objections a reviewer is likely to raise

### 1. Join-level value carrying adds a new path property with no planner owner

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change, ABI / extension API · `maintenance-fit-1`

A later release adds a new join strategy, Append variant or path node and does not propagate or compare the extra values. Nothing fails. The value is recomputed or the cost model credits a value setrefs does not deliver, and users find a slower plan only after upgrading.

Patches 0007 and 0008 make extra values a third axis of path comparison next to cost and pathkeys. They touch add_path(), the new precheck_extras functions, Gather, Material, Memoize, three Append builders and outer-join nullingrels matching. Each future path type has to know about it, and the only check is that the cost side and setrefs agree. No planner committer has commented on 0006-0008, and the entry has no reviewer.

Thread: [#13 (Greg Burd, 2026-10-08)](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#14 (Greg Burd, 2026-10-09)](https://www.postgresql.org/message-id/ChGfmrrDcMPM9quqy69FYxEBqDxgkNG9EgqFal9njID6HIc1O_0E6lmVIbe4wOk-MLR4eGOneBki_awfQIBUlYB6Iz-z5B8BdMfGBkwFGGQ%3D%40burd.me), [#20 (Greg Burd, 2026-10-24)](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me), [#22 (Greg Burd, 2026-11-06)](https://www.postgresql.org/message-id/au4d8NM7cLUMjMd5%40werk)

**Question:** Which committer familiar with add_path() and setrefs.c will own the extra-values property, and what would fail loudly when a new path type ignores it?

Counter-check: held. 0007 and 0008 make extra values a property that add_path(), the prechecks, joins, Gather and Appends must each handle. The material names nothing that fails when a new path type ignores it, and nobody has reviewed these patches.

### 2. Out-of-core GiST and SP-GiST opclasses have no way to opt in

fixable · level 2, rubric item 12 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An extension maintainer ships a GiST opclass whose distance function already returns the operator's exact float8 result, for example a trigram or btree_gist-style distance. With this series the scan still computes the operator again, because only the core point and box opclasses answer yes. Each new opclass needs a core patch and a major release.

The 0002 commit message says GiST answers yes for point_ops and box_ops and that "any out-of-core opclass" defaults to no. Message #1 names btree_gist and pg_trgm among the beneficiaries, and Matthias in message #3 framed this as a per-opclass property. Postgres records opclass capabilities through support functions or opclass properties that an extension can declare. A list of core opclasses inside the AM departs from that practice, and the proposal does not explain why.

Thread: [#1 (Greg Burd, 2026-10-01)](https://www.postgresql.org/message-id/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), [#3 (Matthias van de Meent, 2026-10-05)](https://www.postgresql.org/message-id/CAEze2WjQRfX5cB73JJi__rmnkc1VzXvDSSReXPZN8sBVVKwk5g%40mail.gmail.com), [#4 (Greg Burd, 2026-10-05)](https://www.postgresql.org/message-id/VKFOpl5I4rGZS4ouIc98EAp4AYSVlG7SHpAaV8XHBnr7qA4dTqkeFxMTVBFJ5Nx1V3QTZ-QaykYiPxfg_STJ0QbYtlnZTBiu3BdO7gNkLEA%3D%40burd.me), [#20 (Greg Burd, 2026-10-24)](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me)

**Question:** How does an extension's GiST or SP-GiST opclass declare that its distances are exact, without a change to core?

Counter-check: held. The 0002 commit message says any out-of-core opclass defaults to no, and the callback belongs to the AM. An extension GiST or SP-GiST opclass has no way to answer yes, though message #1 names pg_trgm and btree_gist as cases.

### 3. Earlier advice to wait for index scan unification is not yet settled

fixable · level 2, rubric item 12 · prior rejection, ABI / extension API, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

The amgetbatch rework restructures nodeIndexscan.c and how index scans and index-only scans return values. Whoever commits it must then rework two separate mechanisms from this series: the INNER_VAR slot for ORDER BY values and the index-only scan extra-values path. Neither the amgetbatch authors nor a planner committer has said whether that is a mechanical rebase.

In [P2], waiting for the index scan and index-only scan unification was part of the 2023 feedback, and message #9 confirms this. The series still adds two value paths and a new IndexAmRoutine callback that the unification would likely touch. Message #9 answers that rebasing 0001 is mechanical because the series avoids the AM interface, but 0002 adds amcanreturnorderby to amapi.h. Nobody working on amgetbatch has replied in the thread.

**Precedents.**
- [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (Returned with feedback, 2024-02-10); thread: [link](https://www.postgresql.org/message-id/flat/2ca5865b-4693-40e5-8f78-f3b45d5378fb@iki.fi)

Thread: [#1 (Greg Burd, 2026-10-01)](https://www.postgresql.org/message-id/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), [#9 (Greg Burd, 2026-10-07)](https://www.postgresql.org/message-id/zJpBuDd970r68TPurCEtf9ZZZZIf4HJvNCgY0DKy1qxp9L0ho6ZBy0VoOHOSEZ3K3MmJdMx0dMeqQGLaJ6DnCfKijRydvWK0h58QfL8uKB4%3D%40burd.me)

**Question:** Does someone working on amgetbatch agree that the virtual-slot ORDER BY plumbing and the index-only scan extra values fit the unified scan design?

Counter-check: held. Message #9 answers this only with the author's own view, and message #11 corrects it: the series does add an index AM callback. Nobody working on amgetbatch has replied in the thread.

### 4. Join-level carrying depends on procost, which is mostly a default

fixable · level 2, rubric item 8 · planner behaviour change, performance on the hot path · `prior-attempts-2`

A user has an expression index on a cheap plpgsql or SQL function left at the default COST 100, which passes the 10 times cpu_operator_cost gate. Their eleven-way join then plans about 49% slower and uses more planner memory with no execution gain. Meanwhile an expensive C distance function at COST 1 gets no credit, and the plan stays as before.

[P8] was reverted because "relying on procost to distinguish the relative costs" means little when most functions keep default costs. 0007 uses procost to decide whether a value is carried at all, and the overhead in message #20 is real. Message #7 says a costly function at default COST may not flip its plan, and message #16 says a static COST is not enough. Message #20 points to SupportRequestCost, but few functions have one.

**Precedents.**
- [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (Reverted, 2022-10-03); thread: [#1](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com), [#2](https://www.postgresql.org/message-id/flat/7c79e6a5-8597-74e8-0671-1c39d124c9d6@sigaev.ru), [#3](https://www.postgresql.org/message-id/flat/CA+q6zcW_4o2NC0zutLkOJPsFt80megSpX_dVRo6GK9PC-Jx_Ag@mail.gmail.com)

Thread: [#7 (Greg Burd, 2026-10-06)](https://www.postgresql.org/message-id/yIClNuBkz1PBdI_C5DNORA2WY-hXrhMPUsr4UvSD-AopwswyIlTg_bAJ1p59ewB_qYt31s8hXZrbUaQnkKihoeokq6Q4pY1AKfHDr2Ptt7k%3D%40burd.me), [#13 (Greg Burd, 2026-10-08)](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#16 (Oleg Bartunov, 2026-10-12)](https://www.postgresql.org/message-id/CAF4Au4xgsJGW5p%3DzSUi4C5itYgQXNHCKuogfF3T_gVD4HquU6w%40mail.gmail.com), [#20 (Greg Burd, 2026-10-24)](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me)

**Question:** How many real expression-index and kNN functions carry an accurate procost, and what bounds planning cost when the gate passes for a function whose cost is wrongly declared?

Counter-check: held. A default COST 100 function passes the gate, and message #20 measures +49% planning time at eleven rels. The gate copies the make_sort_input_target() test, which lowers the weight, but the thread shows no bound for a wrongly declared cost.

### 5. Join, outer-join and parallel carrying (0007, 0008) lacks shown user demand

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), planner behaviour change · `demand-1`

A committer takes 0007 and 0008, which add new add_path() dominance rules, extras-aware prechecks and nullingrels relabeling for outer joins, Gather and ordered Appends. Every later planner patch that touches path comparison or join targets must then keep these rules correct, while no reported workload is known to need them.

The demand in the thread covers single-scan kNN reuse: message #3 agrees the double evaluation should be fixed, and message #16 reports a ranked GIN prototype gaining 1.7-2.2x. The join and outer-join cases rest only on constructed queries and on HammerDB mixes built with COST 5000 plpgsql functions (messages #13, #15). Message #14 adds 0008 with the words "I cared", and message #13 asks whether planning cost up to +49% at eleven rels is acceptable without a user case to weigh it against. The planner code is maintained by everyone, but the gain goes to a group the material does not identify.

Thread: [#3 (Matthias van de Meent, 2026-10-05)](https://www.postgresql.org/message-id/CAEze2WjQRfX5cB73JJi__rmnkc1VzXvDSSReXPZN8sBVVKwk5g%40mail.gmail.com), [#13 (Greg Burd, 2026-10-08)](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#14 (Greg Burd, 2026-10-09)](https://www.postgresql.org/message-id/ChGfmrrDcMPM9quqy69FYxEBqDxgkNG9EgqFal9njID6HIc1O_0E6lmVIbe4wOk-MLR4eGOneBki_awfQIBUlYB6Iz-z5B8BdMfGBkwFGGQ%3D%40burd.me), [#15 (Greg Burd, 2026-10-11)](https://www.postgresql.org/message-id/ao9ezhajViiBEVprXjaS8-uLFvpXTHtmS2AJFS_lWCX5cipfOqLHVI2O45LKo05iJ6hB3m03jlp3DlmykV5zrz6F3_noe0GGCXRlU6GMqhs%3D%40burd.me), [#16 (Oleg Bartunov, 2026-10-12)](https://www.postgresql.org/message-id/CAF4Au4xgsJGW5p%3DzSUi4C5itYgQXNHCKuogfF3T_gVD4HquU6w%40mail.gmail.com)

**Question:** Which real queries, from users or from extensions such as ranked GIN, put an expensive index expression below a join, an outer join or a Gather today?

Counter-check: held. Tom's 2023 feedback, quoted in message #9 and recorded in [P2], asked for handling below the top plan level, which supports 0004 and the join part of 0007. For outer joins, Gather and ordered Appends the thread shows only constructed queries.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | 0001 changes <-> and ## results for point and box, so stored values and expression indexes built on them can hold old results after pg_upgrade. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | 0002 adds the IndexAmRoutine callback amcanreturnorderby; 0007 adds add_path_precheck_extras() and new path and plan node fields. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | 0003-0008 change costing, give index paths their own PathTarget and add a dominance rule to add_path(). |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | 0001 changes the results of immutable operators and adds an overflow error in SP-GiST box distance, so it cannot be backpatched. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Extra values become a path property that joins, Gather, Appends, the prechecks and setrefs.c must all keep in agreement. |
| performance on the hot path | **yes** | 0007 adds up to +49% planning time and more planner memory at eleven rels when an expression passes the cost gate. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | For the expression-index cases in 0005-0008, a stored generated column already gives every plan the value. |

## 5. Questions the author must answer before anyone reviews code

1. Which real queries, from users or extensions, put an expensive index expression below an outer join, a Gather or an ordered Append today?
2. What fails loudly when a new path type or join strategy does not propagate or compare the extra values added in 0007 and 0008?
3. How does an extension's GiST or SP-GiST opclass declare that its distances are exact, without a change to core?
4. Does someone working on amgetbatch agree that the INNER_VAR slot for ORDER BY values and the index-only scan extra values fit the unified scan design?
5. What bounds planning time when a cheap function left at the default COST 100 passes the 10 times cpu_operator_cost gate in a many-way join?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows add_path(), PathTarget handling and setrefs.c for 0003-0008, and someone who knows the index AM interface and the amgetbatch work for 0002.
- **Liability:** The owner signs for a new IndexAmRoutine callback that extensions will depend on, and for a new path comparison rule that every later planner patch must respect. Both last for as long as the planner and the AM API exist.
- **Author follow-up:** The author posted thirteen versions in about six weeks and answered review comments in the thread, so follow-up looks likely.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Users of expensive kNN operators and expression indexes gain, and 0001-0003 look separable and lower-risk. The eight-patch series changes add_path(), join targets and setrefs.c, and no planner committer has reviewed 0003-0008.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) Existing plans flip based on declared function COST alone. Not listed: not among the strongest objections.
- `already-exists-1` (fixable, level 1b) Stored generated columns already avoid re-evaluating expensive indexed expressions. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` Point-to-box operator results change for existing data after upgrade. Refuted (already handled): The 0001 commit message lists each changed result and the new SP-GiST overflow error, which GiST and the operator already raise. Correcting an immutable function in a major release is existing practice, and release notes are written from the commit message.
- `maintenance-fit-2` The 10x cpu_operator_cost gate admits every default-cost SQL or PL/pgSQL function. Refuted (not specific): Duplicate of prior-attempts-2. It also misstates COST 100 as 40 times cpu_operator_cost; procost is in units of cpu_operator_cost, so it is 100 times.
- `maintenance-fit-3` Index AM callback shaped around float8 delivery and INNER_VAR reuse. Refuted (wrong facts): The callback takes (opfamily, opcintype, orderbyop) and names no result type; the float4/float8 limit sits in setrefs.c and the executor, so lifting it leaves the callback unchanged. The INNER_VAR slot is executor-internal, and the amgetbatch part duplicates prior-attempts-1.

### Review angles

- prior-attempts: 2 objection(s). The precedent angle finds no fatal objection: the main reasons [P2] was returned are addressed in the series.
- standards-compat: 3 objection(s). The series adds no SQL syntax, GUC or catalog object, so pg_dump round-trip and psql conventions are not affected. The EXPLAIN VERBOSE output changes, but it follows the existing practice for computed scan columns.
- replication-ondisk: 0 objection(s). The series changes only planner and executor behaviour. It adds no on-disk format, no WAL record, no catalog column and no wire protocol message, so pg_upgrade, standbys and subscribers are not affected. Adding a NULL-default IndexAmRoutine callback and new plan node fields is a normal major-version ABI change. One persistent effect is minor and needs only a release-note item: 0001 changes <-> and ## results for points near a box edge, so existing expression indexes or stored generated columns built on those operators can hold old values after pg_upgrade until they are reindexed or recomputed. For the same reason 0001 should not be backpatched.
- security: 0 objection(s). No security objection. The series changes no privilege check and adds no new object, role or setting, and it lets no one see data they could not already select. The substituted value is the result of an immutable index expression or ordering operator for a row that already passed every qual. The planner memory growth at deep joins is bounded by query size, like ordinary join search.
- maintenance-fit: 3 objection(s). Patches 0001 to 0003 look separable and lower-risk. The objections concern mostly 0007 and 0008 and the long-term API.
- already-exists: 1 objection(s). For 0002-0004, the reuse of non-rechecked kNN ORDER BY values, no existing core feature does this. The only extension workaround, a backend global, breaks with nested ordering scans (message #9), so this angle raises nothing there.
- demand: 1 objection(s). Demand for the base change (0001-0003) is shown and has lasted: earlier threads, Matthias in message #3 and Oleg's ranked GIN test in message #16. The one objection covers only the planner extensions in 0007-0008.

### Precedents retrieved

- P1: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P2: [Avoid computing ORDER BY junk columns unnecessarily](https://commitfest.postgresql.org/patch/4717/) (cf, Returned with feedback, 2024-02-10) - used in the review; reason summarised from its thread
- P3: [Index Only Scan for index expressions](https://commitfest.postgresql.org/patch/738/) (cf, Returned with feedback, 2016-09-28) - used in the review; reason summarised from its thread
- P4: [Index Only Scan support for cube](https://commitfest.postgresql.org/patch/1146/) (cf, Rejected, 2017-09-20) - judged not relevant; reason summarised from its thread
- P5: [basebackup: support zstd long distance matching](https://commitfest.postgresql.org/patch/4111/) (cf, Withdrawn, 2023-04-04) - judged not relevant; reason summarised from its thread
- P6: [Fix buffer pinning logic/visibility bugs in \[SP-\]Gist Index-Only Scans](https://commitfest.postgresql.org/patch/5542/) (cf, Withdrawn, 2026-06-08) - judged not relevant; reason summarised from its thread
- P7: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P8: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - used in the review
- P9: [Fix ALTER TABLE DROP EXPRESSION with ONLY option](https://commitfest.postgresql.org/patch/5992/) (cf, Returned with feedback, 2026-04-02) - judged not relevant; reason summarised from its thread
- P10: [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - used in the review
- P11: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P12: [Support SK_SEARCHNULL / SK_SEARCHNOTNULL for heap-only scans](https://commitfest.postgresql.org/patch/4180/) (cf, Returned with feedback, 2023-12-02) - judged not relevant; reason summarised from its thread
- P13: [Revert buggy optimization of index scans](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c7111d11b188bd631a63c98e9aab8532176007c3) (revert, Reverted, 2016-03-03) - judged not relevant
- P14: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - judged not relevant
- P15: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [apply_scanjoin_target_to_paths and partitionwise join](https://commitfest.postgresql.org/patch/4929/) (cf, Returned with feedback, 2025-01-13) - judged not relevant
- P17: [JIT expression evaluation improvements](https://commitfest.postgresql.org/patch/2501/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P18: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P19: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P20: [support for box type in SP-GiST index](https://commitfest.postgresql.org/patch/419/) (cf, Returned with feedback, 2016-01-28) - judged not relevant
- P21: [Add parallel columns for seq scan and index scan on pg_stat_all_tables and _indexes](https://commitfest.postgresql.org/patch/5238/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P22: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant
- P23: [Functions for sorting GiST build of gist_btree types](https://commitfest.postgresql.org/patch/2824/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P24: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant

### Warnings

- attachment limit (8) reached; v1-0001-Let-an-ordering-index-scan-hand-its-ORDER-BY-valu.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 223 s; cost reported by the CLI: $4.76
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 105 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:50:12.115Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
