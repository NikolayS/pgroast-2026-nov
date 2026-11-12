# Design pre-review: "variable not found in subplan target list", DISTINCT agg over Gather`

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7600](https://commitfest.postgresql.org/patch/7600/); [thread](https://www.postgresql.org/message-id/flat/19804-594c35d8647fd7ef%40postgresql.org), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real, reproducible on 16 through master, and the patch follows the direction suggested in message #3. Two fixable level 2 points survive, and neither argues against the fix. standards-compat-1 asks which working plans the stricter relation_can_be_sorted_early() check loses, and replication-ondisk-1 asks what the paths.h change means for back branches. Both need a committer who knows the planner to settle scope per branch.

Not checked: Without the code, the review cannot check the rules for independently supplied Vars, PHVs and nullingrels, or the Append and MergeAppend column-position checks. It also cannot check which paths.h declarations change.

## Prior attempts

None found before 2026-10-28.

## Objections a reviewer is likely to raise

### 1. Conservative early-sort check may change plans for working queries

A user on 16.x has a parallel query whose plan sorts early below a Gather, and the query does not hit the bug. After a minor upgrade, relation_can_be_sorted_early() rejects that sort, so the planner picks a slower plan. The user finds out from latency, with no error and no setting to undo it.

Message #4 says the new fallback 'might reject some otherwise valid early-sort opportunities', and no measure of the lost plans exists. The bug spans 16 through master, so a fix will likely be backpatched, and in a back branch a plan change for queries that are not broken is a compatibility cost. The proposal does not separate the narrow crash fix from the broader tightening, or say which branches each part should go to.

**Question:** Which queries that plan and run correctly today lose an early sort under the new check, and can that part stay out of the back branches?

<sub>fixable, level 2, planner behaviour change; messages [#4](https://www.postgresql.org/message-id/CAHewXN%3Dw6BjEHDfkC5ZTzc35FHHDc%3DKE8Nkw%3DYj_3_K-VvVRNw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAHewXNn3dOxH9qf9yJrdrbBVxRy%3DkDMFjwdZ%2BvZOK6%3Dq-_PUzQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Back-branch ABI effect of the paths.h change is not stated

The fix goes into a 16.x or 17.x minor release with a changed declaration in paths.h. An extension built against the earlier minor release keeps calling the old signature. It may crash or misplan after the server is updated in place, with no rebuild step to warn the user.

The bug exists on 16 through master (message #1), so the fix needs back-patching, but the patch prose does not name target branches. The patch touches src/include/optimizer/paths.h, and the brief lists a signature change that extensions can see. The commit message keeps find_computable_ec_member() unchanged but does not say which exported declaration changes. A committer would likely ask for a separate back-branch variant that keeps existing signatures, for example a new function beside the old one.

**Question:** Which exported declarations in paths.h change, and what form of the fix keeps them ABI-compatible on 16 through 19?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/19804-594c35d8647fd7ef%40postgresql.org), [#4](https://www.postgresql.org/message-id/CAHewXN%3Dw6BjEHDfkC5ZTzc35FHHDc%3DKE8Nkw%3DYj_3_K-VvVRNw%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAHewXNn3dOxH9qf9yJrdrbBVxRy%3DkDMFjwdZ%2BvZOK6%3Dq-_PUzQ%40mail.gmail.com); `replication-ondisk-1`</sub>

## Questions for the author

1. Which queries that plan and run correctly today lose an early sort under the new relation_can_be_sorted_early() check?
2. Which branches does the patch target, and does the stricter relation_can_be_sorted_early() check go to the back branches or only to master?
3. Which declarations in paths.h change, and does the back-branch form of the fix keep existing exported signatures?
4. Why is the output-side nullingrels rule for outer-join inputs correct for choosing a sort EC member?
5. Which local PHVs does the input list keep, and what case shows that each kept PHV is available to the projection?

## Touch points

- **ABI / extension API:** The patch touches src/include/optimizer/paths.h, a header that extensions include.
- **planner behaviour change:** The patch changes sort EC member choice, Append and MergeAppend plan creation, and the relation_can_be_sorted_early() check.
- **backpatchability:** The bug exists on 16 through master, and the patch names no target branches.
- **maintenance burden ("who maintains this in 10 years"):** createplan.c gains rules for which child outputs count as independently supplied Vars, PHVs and nestloop parameters.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane diagnosed the cause in message #3 and suggested the caller-supplied input list. No committer has reviewed the WIP patch or v2 yet.
- Priority: proceed; audience niche; review cost medium. A valid query fails with an internal error on all supported branches from 16 up, though the trigger needs a rare plan shape found by a fuzzer. The patch is about 35 KiB in createplan.c and equivclass.c, and review needs someone who knows EC and setrefs behaviour.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
