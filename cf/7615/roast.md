# Design pre-review: Eliminate redundant outer-join clone clauses after constant propagation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7615](https://commitfest.postgresql.org/patch/7615/); [thread](https://www.postgresql.org/message-id/flat/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The proposal does not name a target branch, and the answer decides the design (standards-compat-1). ABI stability is the only reason given to avoid the v3 RestrictInfo field, and that limit applies only to back branches. Message #1 also shows no query or timing that got worse after f4002899 (demand-1). Both are fixable, but a committer would likely ask for them before reading the nullability check.

Not checked: Without the code, it could not be judged whether the nullability check covers every deferred pushed-down outer join case or what it costs at planning time. The two level 2 objections on norm_selec reuse and the duplicated nullability reasoning were not evaluated.

## Prior attempts

- [Redundant qualifier elimination](https://www.postgresql.org/message-id/flat/DKBX1KUW8KUW.2DU5RD192DHIQ@gmail.com) (related; Returned with feedback, 2026-08-14). "It would impose nontrivial costs on every query --- roughly O(N^2) in the number of restriction clauses --- in order to fix badly-written queries, and that is not a tradeoff I like." ([message](https://www.postgresql.org/message-id/3766950.1786724867%40sss.pgh.pa.us)) Different now: P4 compared every base restriction clause against all others with equal() for every query. This patch tests only clone clauses already marked norm_selec > 1 when it builds a join or parameterized-path restriction list.

## Objections a reviewer is likely to raise

### 1. Plan-changing optimization framed around back-branch ABI limits

A user on a stable branch applies a minor release that contains this patch. A query with outer joins and a constant-propagated equality gets a different join order and runs slower. The user finds out from production latency after an upgrade that was expected to be safe.

Message #1 gives ABI stability as the only reason to avoid the v3 RestrictInfo field, and that constraint applies only to back-branches. The proposal does not say whether it targets master or stable branches. Postgres does not usually back-patch planner optimizations that change plans, and join.out shows the plans change. On master, a new field is allowed, so the ABI argument for this design does not hold there.

**Question:** Is this patch meant for back-branches, and if it is only for master, why not use the clearer v3 RestrictInfo field instead?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com); `standards-compat-1`</sub>

### 2. No evidence the leftover join filters cost any user anything

A committer adds a second, subtle nullability check to clause selection in relnode.c, in the area where f4002899 just fixed a wrong-result bug. The only gain is dropping a redundant equality filter, because the derived INNERVAR = CONSTANT already applies. If the new check misses a commuting-join case, users get wrong results, and they find out from bug reports.

Message #1 gives no field report, no -general or -bugs thread and no query that got slower after f4002899. It also gives no execution or planning-time numbers. The only evidence is changed plans in join.out, which show the plan differs but not that it is faster. With f4002899, the leftover clause adds no selectivity, so the main remaining cost is one cheap filter per row.

**Question:** Which real queries got measurably slower after f4002899, and by how much does this patch improve their planning or execution time?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAHewXNk8Xpn7RTZFK2EeqTxe10Nmg8eQz3qj3i2FSmNFce1y_Q%40mail.gmail.com); `demand-1`</sub>

## Questions for the author

1. Is this patch meant for back branches or for master only?
2. If it targets master only, why is reusing norm_selec > 1 better than the explicit RestrictInfo field from v3?
3. Which queries got measurably slower after f4002899, and how much does this patch improve their planning or execution time?
4. Which regression test exercises the deferred pushed-down outer join case, so that a later commutation change that breaks the check fails visibly?
5. What stops other planner code from setting norm_selec above 1 on a clause that must still be evaluated?

## Touch points

- **ABI / extension API:** The design avoids a new RestrictInfo field for ABI reasons and gives norm_selec > 1 a second meaning.
- **planner behaviour change:** join.out shows dropped join filters and different join orders for outer-join queries.
- **backpatchability:** Message #1 presents ABI safety as the gain over v3 but does not name a target branch.
- **maintenance burden ("who maintains this in 10 years"):** relnode.c gains a separate outer-join nullability check built from six RestrictInfo and SpecialJoinInfo fields.
- **performance on the hot path:** The check runs when the planner builds join and parameterized-path restriction lists; no planning-time numbers are given.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows outer-join commutation, clone clauses and incompatible_relids handling in the planner. No one has replied to message #1.
- Priority: fix design first; audience niche; review cost medium. The patch is small and touches one planner file, but it sits where f4002899 just fixed a wrong-result bug. The target branch must be settled first, because it decides whether reusing norm_selec is justified.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
