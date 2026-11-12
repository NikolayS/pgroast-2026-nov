# Design pre-review: Let an ordering index scan hand its ORDER BY value to the target list

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7392](https://commitfest.postgresql.org/patch/7392/); [thread](https://www.postgresql.org/message-id/flat/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), 22 of 22 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal has support: message #3 agrees the double evaluation should be fixed, and the series answers the main complaints recorded in [P2]. Three design points are still open. maintenance-fit-1: 0007 and 0008 add a path property that nothing checks. standards-compat-1: extension opclasses cannot opt in. prior-attempts-1: nobody working on amgetbatch has said the series fits it. demand-1 and prior-attempts-2 put the cost of 0007-0008 in question. Nothing that survived is fatal.

Not checked: Without the code, the agreement between path_target_cost() and the setrefs.c rewrite cannot be checked. The same holds for the outer-join nullingrels matching in 0008 and the EvalPlanQual and reorder-queue value handling in 0002.

## Prior attempts

- [Avoid computing ORDER BY junk columns unnecessarily](https://www.postgresql.org/message-id/flat/2ca5865b-4693-40e5-8f78-f3b45d5378fb@iki.fi) (similar idea; Returned with feedback, 2024-02-10). "My basic complaint is that this is a band-aid that only touches things at a surface level, whereas I think we need a much deeper set of changes in order to have a plausible solution." ([message](https://www.postgresql.org/message-id/2935085.1703806977%40sss.pgh.pa.us)) Different now: P2 removed junk columns only in the final targetlist and changed resjunk to an enum. This series leaves resjunk alone, fixes costing (0003, 0005) and carries values below joins (0004, 0007). The advice to wait for index scan unification is still open.
- [Index Only Scan for index expressions](https://www.postgresql.org/message-id/flat/57B23C4F.7000304@postgrespro.ru) (similar idea; Returned with feedback, 2016-09-28). "Based on this report, this patch appears to have bugs that would preclude committing it, so I'm marking it "Returned with Feedback" for this CommitFest, which is due to end shortly." ([message](https://www.postgresql.org/message-id/CA%2BTgmoaiYuarPYv2BWrXG-7XZ1EX75guirx2kTXQY0HxtD%3DWuw%40mail.gmail.com)) Different now: P3 added index-only scan matching for expression columns and was returned because of bugs. This series keeps the existing matching and changes costing and path targets (0005, 0006).
- [Improve use of expression indexes for ORDER BY](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (similar idea; Not worth doing, date unknown). reason not recorded Different now: The row lists a similar goal as not worth doing but records no reason. This series brings measurements and a ranked GIN test from message #16.

## Objections a reviewer is likely to raise

### 1. Join-level value carrying adds a new path property with no planner owner

A later release adds a new join strategy, Append variant or path node and does not propagate or compare the extra values. Nothing fails. The value is recomputed or the cost model credits a value setrefs does not deliver, and users find a slower plan only after upgrading.

Patches 0007 and 0008 make extra values a third axis of path comparison next to cost and pathkeys. They touch add_path(), the new precheck_extras functions, Gather, Material, Memoize, three Append builders and outer-join nullingrels matching. Each future path type has to know about it, and the only check is that the cost side and setrefs agree. No planner committer has commented on 0006-0008, and the entry has no reviewer.

**Question:** Which committer familiar with add_path() and setrefs.c will own the extra-values property, and what would fail loudly when a new path type ignores it?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#13](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#14](https://www.postgresql.org/message-id/ChGfmrrDcMPM9quqy69FYxEBqDxgkNG9EgqFal9njID6HIc1O_0E6lmVIbe4wOk-MLR4eGOneBki_awfQIBUlYB6Iz-z5B8BdMfGBkwFGGQ%3D%40burd.me), [#20](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me), [#22](https://www.postgresql.org/message-id/au4d8NM7cLUMjMd5%40werk); `maintenance-fit-1`</sub>

### 2. Out-of-core GiST and SP-GiST opclasses have no way to opt in

An extension maintainer ships a GiST opclass whose distance function already returns the operator's exact float8 result, for example a trigram or btree_gist-style distance. With this series the scan still computes the operator again, because only the core point and box opclasses answer yes. Each new opclass needs a core patch and a major release.

The 0002 commit message says GiST answers yes for point_ops and box_ops and that "any out-of-core opclass" defaults to no. Message #1 names btree_gist and pg_trgm among the beneficiaries, and Matthias in message #3 framed this as a per-opclass property. Postgres records opclass capabilities through support functions or opclass properties that an extension can declare. A list of core opclasses inside the AM departs from that practice, and the proposal does not explain why.

**Question:** How does an extension's GiST or SP-GiST opclass declare that its distances are exact, without a change to core?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), [#3](https://www.postgresql.org/message-id/CAEze2WjQRfX5cB73JJi__rmnkc1VzXvDSSReXPZN8sBVVKwk5g%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/VKFOpl5I4rGZS4ouIc98EAp4AYSVlG7SHpAaV8XHBnr7qA4dTqkeFxMTVBFJ5Nx1V3QTZ-QaykYiPxfg_STJ0QbYtlnZTBiu3BdO7gNkLEA%3D%40burd.me), [#20](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me); `standards-compat-1`</sub>

### 3. Earlier advice to wait for index scan unification is not yet settled

The amgetbatch rework restructures nodeIndexscan.c and how index scans and index-only scans return values. Whoever commits it must then rework two separate mechanisms from this series: the INNER_VAR slot for ORDER BY values and the index-only scan extra-values path. Neither the amgetbatch authors nor a planner committer has said whether that is a mechanical rebase.

In [P2], waiting for the index scan and index-only scan unification was part of the 2023 feedback, and message #9 confirms this. The series still adds two value paths and a new IndexAmRoutine callback that the unification would likely touch. Message #9 answers that rebasing 0001 is mechanical because the series avoids the AM interface, but 0002 adds amcanreturnorderby to amapi.h. Nobody working on amgetbatch has replied in the thread.

Precedent: [Avoid computing ORDER BY junk columns unnecessarily](https://www.postgresql.org/message-id/flat/2ca5865b-4693-40e5-8f78-f3b45d5378fb@iki.fi) (Returned with feedback, 2024-02-10)

**Question:** Does someone working on amgetbatch agree that the virtual-slot ORDER BY plumbing and the index-only scan extra values fit the unified scan design?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/8s5lT8erXzBMugXJQ6Wginbp_gc2B4hmCYhF0Q0GpnuK87eCAOcKs5K31AWCwraCNFIQt42wwkow_MPPubHg485MWv-zDBY4NL_JX9yJEEg%3D%40burd.me), [#9](https://www.postgresql.org/message-id/zJpBuDd970r68TPurCEtf9ZZZZIf4HJvNCgY0DKy1qxp9L0ho6ZBy0VoOHOSEZ3K3MmJdMx0dMeqQGLaJ6DnCfKijRydvWK0h58QfL8uKB4%3D%40burd.me); `prior-attempts-1`</sub>

### 4. Join-level carrying depends on procost, which is mostly a default

A user has an expression index on a cheap plpgsql or SQL function left at the default COST 100, which passes the 10 times cpu_operator_cost gate. Their eleven-way join then plans about 49% slower and uses more planner memory with no execution gain. Meanwhile an expensive C distance function at COST 1 gets no credit, and the plan stays as before.

[P8] was reverted because "relying on procost to distinguish the relative costs" means little when most functions keep default costs. 0007 uses procost to decide whether a value is carried at all, and the overhead in message #20 is real. Message #7 says a costly function at default COST may not flip its plan, and message #16 says a static COST is not enough. Message #20 points to SupportRequestCost, but few functions have one.

Precedent: [Revert "Optimize order of GROUP BY keys".](https://www.postgresql.org/message-id/flat/TYAPR01MB586665EB5FB2C3807E893941F5579@TYAPR01MB5866.jpnprd01.prod.outlook.com) (Reverted, 2022-10-03)

**Question:** How many real expression-index and kNN functions carry an accurate procost, and what bounds planning cost when the gate passes for a function whose cost is wrongly declared?

<sub>fixable, level 2, planner behaviour change; messages [#7](https://www.postgresql.org/message-id/yIClNuBkz1PBdI_C5DNORA2WY-hXrhMPUsr4UvSD-AopwswyIlTg_bAJ1p59ewB_qYt31s8hXZrbUaQnkKihoeokq6Q4pY1AKfHDr2Ptt7k%3D%40burd.me), [#13](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#16](https://www.postgresql.org/message-id/CAF4Au4xgsJGW5p%3DzSUi4C5itYgQXNHCKuogfF3T_gVD4HquU6w%40mail.gmail.com), [#20](https://www.postgresql.org/message-id/4NbzZrWHTdxBYk1yMvPE2r7eDi1M9-qvp9LuO8b145nKv6M3S5A7zHl3LhM0wW2O4Yy6_HWr_6HF7iaAGtLTjRaICJ8h_IyM734HQQFZSYc%3D%40burd.me); `prior-attempts-2`</sub>

### 5. Join, outer-join and parallel carrying (0007, 0008) lacks shown user demand

A committer takes 0007 and 0008, which add new add_path() dominance rules, extras-aware prechecks and nullingrels relabeling for outer joins, Gather and ordered Appends. Every later planner patch that touches path comparison or join targets must then keep these rules correct, while no reported workload is known to need them.

The demand in the thread covers single-scan kNN reuse: message #3 agrees the double evaluation should be fixed, and message #16 reports a ranked GIN prototype gaining 1.7-2.2x. The join and outer-join cases rest only on constructed queries and on HammerDB mixes built with COST 5000 plpgsql functions (messages #13, #15). Message #14 adds 0008 with the words "I cared", and message #13 asks whether planning cost up to +49% at eleven rels is acceptable without a user case to weigh it against. The planner code is maintained by everyone, but the gain goes to a group the material does not identify.

**Question:** Which real queries, from users or from extensions such as ranked GIN, put an expensive index expression below a join, an outer join or a Gather today?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/CAEze2WjQRfX5cB73JJi__rmnkc1VzXvDSSReXPZN8sBVVKwk5g%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/O7LiKSyMJH9pKrU0jxstopxs00AioGMev0lpPwq2uhUwzurMTQBFUKOdFPq0LXd4GHxxrIpajWq9Vy8QwkCsb3LgJrCdgfU7fn36dc4FFWQ%3D%40burd.me), [#14](https://www.postgresql.org/message-id/ChGfmrrDcMPM9quqy69FYxEBqDxgkNG9EgqFal9njID6HIc1O_0E6lmVIbe4wOk-MLR4eGOneBki_awfQIBUlYB6Iz-z5B8BdMfGBkwFGGQ%3D%40burd.me), [#15](https://www.postgresql.org/message-id/ao9ezhajViiBEVprXjaS8-uLFvpXTHtmS2AJFS_lWCX5cipfOqLHVI2O45LKo05iJ6hB3m03jlp3DlmykV5zrz6F3_noe0GGCXRlU6GMqhs%3D%40burd.me), [#16](https://www.postgresql.org/message-id/CAF4Au4xgsJGW5p%3DzSUi4C5itYgQXNHCKuogfF3T_gVD4HquU6w%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Which real queries, from users or extensions, put an expensive index expression below an outer join, a Gather or an ordered Append today?
2. What fails loudly when a new path type or join strategy does not propagate or compare the extra values added in 0007 and 0008?
3. How does an extension's GiST or SP-GiST opclass declare that its distances are exact, without a change to core?
4. Does someone working on amgetbatch agree that the INNER_VAR slot for ORDER BY values and the index-only scan extra values fit the unified scan design?
5. What bounds planning time when a cheap function left at the default COST 100 passes the 10 times cpu_operator_cost gate in a many-way join?

## Touch points

- **on-disk format / pg_upgrade:** 0001 changes <-> and ## results for point and box, so stored values and expression indexes built on them can hold old results after pg_upgrade.
- **ABI / extension API:** 0002 adds the IndexAmRoutine callback amcanreturnorderby; 0007 adds add_path_precheck_extras() and new path and plan node fields.
- **planner behaviour change:** 0003-0008 change costing, give index paths their own PathTarget and add a dominance rule to add_path().
- **backpatchability:** 0001 changes the results of immutable operators and adds an overflow error in SP-GiST box distance, so it cannot be backpatched.
- **maintenance burden ("who maintains this in 10 years"):** Extra values become a path property that joins, Gather, Appends, the prechecks and setrefs.c must all keep in agreement.
- **performance on the hot path:** 0007 adds up to +49% planning time and more planner memory at eleven rels when an expression passes the cost gate.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** For the expression-index cases in 0005-0008, a stored generated column already gives every plan the value.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows add_path(), PathTarget handling and setrefs.c for 0003-0008, and someone who knows the index AM interface and the amgetbatch work for 0002.
- Priority: needs champion; audience a sizeable minority; review cost high. Users of expensive kNN operators and expression indexes gain, and 0001-0003 look separable and lower-risk. The eight-patch series changes add_path(), join targets and setrefs.c, and no planner committer has reviewed 0003-0008.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
