# Design pre-review: Mark the enum comparison functions as leakproof

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6752](https://commitfest.postgresql.org/patch/6752/); [thread](https://www.postgresql.org/message-id/flat/8222d01be22d9d87ba1e7f47e398f4ce81af4aaf.camel%40cybertec.at), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The reported need is enum equality only (messages #4 and #5), and message #3 already accepts enum_eq as safe. demand-1 and demand-2 show that enum_cmp and the ordering operators have no user case and no measurement, while message #3 sets a performance bar for them. The list has to agree on the scope before code review makes sense. Only 3 of 5 messages were visible in full, so confidence is medium.

Not checked: The code was not read, so the set of functions the patch marks and the error paths listed in message #1 are unchecked. Level 2 objections on the narrower threat model and on future changes to enum_cmp were not evaluated.

## Prior attempts

- [Row Level Security, leakproof-ness, and performance implications](https://www.postgresql.org/message-id/flat/2811772.0XtDgEdalL@peanuts2) (similar idea; Rejected, 2019-03-19). "I think that approach has been found not to actually satisfy the leakproofness criteria." ([message](https://www.postgresql.org/message-id/a5f0f8ec-bcdb-154b-be27-4109ee18edfa%402ndquadrant.com)) Different now: P1's patch marked enum_eq and enum_ne, and its thread also discussed redacting error messages and a GUC to treat all functions as leakproof. This proposal adds enum_cmp, uses no redaction, and argues that the reachable errors reveal no compared values.

## Objections a reviewer is likely to raise

### 1. Demand covers enum equality only, not enum_cmp or ordering operators

The patch marks enum_cmp and the ordering operators leakproof even though no user in the thread needs them. Later changes to the enum cache or catalog lookup code then carry a leakproof obligation that a reviewer can miss. If such a change adds an error that leaks, RLS deployments with enums are exposed for a benefit nobody asked for.

The only concrete user names enum_eq as the function that blocked index conditions (message #5), and the author confirms that "only equality comparisons are important" (message #4). Tom Lane objects to enum_cmp only and accepts enum_eq (message #3). The one argument for including enum_cmp is that it can stand in for enum_eq, and the thread does not answer it. As things stand, the riskier half of the patch has no demand behind it.

**Question:** Which real query needs a range or ordering enum predicate as an index condition under RLS, and why would marking only enum_eq not serve it?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/3652798.1782945225%40sss.pgh.pa.us), [#4](https://www.postgresql.org/message-id/6447036b558f6c4e1cf3d695e5682c97cc3c270c.camel%40cybertec.at), [#5](https://www.postgresql.org/message-id/CAMxA3rvwEgfbBqRx6UB2TOAmnLp5-SDLmCuhtvKo24i6iY%2Ba2g%40mail.gmail.com); `demand-1`</sub>

### 2. No measurement shows the cost clears the bar used for text_cmp

A committer weighs the risk of enum_cmp against its benefit and finds no number for the benefit. The only user in the thread reports that workarounds exist for every function on his list. The leakproof promise, which is hard to take back, then rests on the needs of one customer.

Tom Lane says text_cmp was an exception because the performance cost of not marking it was unacceptable (message #3). The thread gives no benchmark, plan, or slowdown figure for enum predicates under RLS. Andreas Lind writes that "we eventually found workarounds for all of these issues" (message #5). Demand has lasted since at least the earlier attempt on enum_eq [P1], which supports equality but does not measure the cost for ordering operators.

Precedent: [Row Level Security, leakproof-ness, and performance implications](https://www.postgresql.org/message-id/flat/2811772.0XtDgEdalL@peanuts2) (Rejected, 2019-03-19)

**Question:** What slowdown do users see with enum ordering predicates under RLS, measured on a real schema, and why is the workaround Andreas Lind used not acceptable?

<sub>fixable, level 1b, demand; messages [#3](https://www.postgresql.org/message-id/3652798.1782945225%40sss.pgh.pa.us), [#4](https://www.postgresql.org/message-id/6447036b558f6c4e1cf3d695e5682c97cc3c270c.camel%40cybertec.at), [#5](https://www.postgresql.org/message-id/CAMxA3rvwEgfbBqRx6UB2TOAmnLp5-SDLmCuhtvKo24i6iY%2Ba2g%40mail.gmail.com); `demand-2`</sub>

## Questions for the author

1. Would a patch that marks only enum_eq and enum_ne leakproof meet the need reported in messages #4 and #5?
2. In what concrete query or plan does enum_cmp stand in for enum_eq, so that enum_cmp must be leakproof for equality predicates to work under RLS?
3. What slowdown does a real schema show for enum ordering predicates under RLS, and why are the workarounds mentioned in message #5 not acceptable?
4. Can the case for the enum functions be argued under the existing leakproof rule, where the restricted role runs arbitrary SQL and OOM errors count?
5. Which functions does the patch mark leakproof, by name?

## Touch points

- **catalog changes:** The patch changes proleakproof for the enum comparison functions in pg_proc.dat.
- **planner behaviour change:** The planner can push enum comparisons below RLS quals and security barrier views, including as index conditions.
- **security / privileges:** A leakproof marking is a security promise for RLS and security barrier views, and message #1 proposes a narrower threat model.
- **backpatchability:** The change edits pg_proc.dat, so it cannot go to released branches.
- **maintenance burden ("who maintains this in 10 years"):** Every later change to the enum cache and catalog lookup code reached by enum_cmp must keep it leakproof, with no test.
- **performance on the hot path:** see objection demand-2: No measurement shows the cost clears the bar used for text_cmp

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane replied in message #3: he accepts enum_eq as safe and wants enum_cmp rejected. A committer who knows the enum cache and catcache code would have to sign for enum_cmp.
- Not now, show demand first: For enum_cmp and the ordering operators: a concrete case where enum_cmp is used in place of enum_eq under RLS, or a measured slowdown on a real schema with enum ordering predicates that the known workarounds do not solve. The enum_eq part needs no further evidence of demand.
- Priority: fix design first; audience niche; review cost low. The patch is a 3.4 KB catalog change in two files, and the users are sites that combine RLS with enum columns. The burden is low for enum_eq and higher for enum_cmp, so a version limited to equality, or one with a shown need for enum_cmp, could proceed.

---

[roast-full.md](roast-full.md) has 2 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
