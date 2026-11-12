# Design pre-review: Mark the enum comparison functions as leakproof

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6752](https://commitfest.postgresql.org/patch/6752/) (Needs review); authors: Laurenz Albe (laurenz)
- Thread: [Making the ENUM operators LEAKPROOF](https://www.postgresql.org/message-id/flat/8222d01be22d9d87ba1e7f47e398f4ce81af4aaf.camel%40cybertec.at), started 2026-04-29
- Messages read: 5 of 5
- Patch prose read: v1-0001-Mark-the-enum-comparison-functions-as-leakproof.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The reported need is enum equality only (messages #4 and #5), and message #3 already accepts enum_eq as safe. demand-1 and demand-2 show that enum_cmp and the ordering operators have no user case and no measurement, while message #3 sets a performance bar for them. The list has to agree on the scope before code review makes sense. Only 3 of 5 messages were visible in full, so confidence is medium.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the set of functions the patch marks and the error paths listed in message #1 are unchecked. Level 2 objections on the narrower threat model and on future changes to enum_cmp were not evaluated.

## The proposal, restated

**Problem.** Under row-level security, predicates on enum columns cannot be used as index conditions because enum_eq and enum_cmp are not leakproof. Users with enums in indexes and composite primary keys see severe performance regressions once they enable RLS.

**Proposed design.** The patch sets the leakproof flag on the enum comparison functions in pg_proc.dat and updates the opr_sanity regression output. The author argues that the only side channel is error messages, and that none of the reachable errors reveal the compared values. He also proposes that a leakproof proof only needs to consider attacks through an application, not arbitrary SQL, and may ignore out-of-memory errors.

**User-visible surface.**
- enum_eq, enum_cmp and the other enum comparison functions show proleakproof = true in pg_proc
- Planner can push enum comparisons into security barrier views and RLS quals, including as index conditions

**What the author claims.**
- Errors reachable from enum_cmp_internal() occur only for non-enum input or data corruption, which SQL cannot trigger.
- Cache memory allocations reveal only pg_enum contents, which are already public.
- Out-of-memory errors that show allocation sizes cannot be exploited through an application.
- Text comparison is already leakproof and has a larger detoast OOM risk than enum_cmp.
- A customer depends on enum comparisons working with RLS.

**Thread state.** Tom Lane wants to reject enum_cmp because its code is too large to confirm safe now and after future changes, but he agrees enum_eq is safe. Laurenz says only equality matters to the user but cites an earlier argument that enum_cmp can stand in for enum_eq; nobody has answered that point.

## 2. Prior attempts

### Row Level Security, leakproof-ness, and performance implications (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2038/](https://commitfest.postgresql.org/patch/2038/)
- **Thread:** [https://www.postgresql.org/message-id/flat/2811772.0XtDgEdalL@peanuts2](https://www.postgresql.org/message-id/flat/2811772.0XtDgEdalL@peanuts2)
- **Date:** decided 2019-03-19; started 2019-02-19
- **Outcome:** Rejected (cf)
- **Who objected:** Peter Eisentraut, Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Peter Eisentraut wrote that redacting error messages had been found not to satisfy the leakproofness criteria, and Joe Conway accepted that the consensus was against him. Robert Haas suggested doing it outside core, with a logging hook and functions marked leakproof by hand, and doubted that customers could be supported with redacted messages. Peter also found the CREATE POLICY page the wrong place for the documentation patch and asked how a list of non-leakproof operators would be kept complete. The visible messages do not show which of these two patches this entry is.
- **Reason (quote):** "I think that approach has been found not to actually satisfy the leakproofness criteria." ([message](https://www.postgresql.org/message-id/a5f0f8ec-bcdb-154b-be27-4109ee18edfa%402ndquadrant.com))
- **What is different this time:** P1's patch marked enum_eq and enum_ne, and its thread also discussed redacting error messages and a GUC to treat all functions as leakproof. This proposal adds enum_cmp, uses no redaction, and argues that the reachable errors reveal no compared values.


## 3. Objections a reviewer is likely to raise

### 1. Demand covers enum equality only, not enum_cmp or ordering operators

fixable · level 1b, rubric item 3 · demand, security / privileges, maintenance burden ("who maintains this in 10 years") · `demand-1`

The patch marks enum_cmp and the ordering operators leakproof even though no user in the thread needs them. Later changes to the enum cache or catalog lookup code then carry a leakproof obligation that a reviewer can miss. If such a change adds an error that leaks, RLS deployments with enums are exposed for a benefit nobody asked for.

The only concrete user names enum_eq as the function that blocked index conditions (message #5), and the author confirms that "only equality comparisons are important" (message #4). Tom Lane objects to enum_cmp only and accepts enum_eq (message #3). The one argument for including enum_cmp is that it can stand in for enum_eq, and the thread does not answer it. As things stand, the riskier half of the patch has no demand behind it.

Thread: [#3 (Tom Lane, 2026-07-01)](https://www.postgresql.org/message-id/3652798.1782945225%40sss.pgh.pa.us), [#4 (Laurenz Albe, 2026-07-20)](https://www.postgresql.org/message-id/6447036b558f6c4e1cf3d695e5682c97cc3c270c.camel%40cybertec.at), [#5 (Andreas Lind, 2026-07-22)](https://www.postgresql.org/message-id/CAMxA3rvwEgfbBqRx6UB2TOAmnLp5-SDLmCuhtvKo24i6iY%2Ba2g%40mail.gmail.com)

**Question:** Which real query needs a range or ordering enum predicate as an index condition under RLS, and why would marking only enum_eq not serve it?

Counter-check: held. Messages #4 and #5 confirm that the reported need is equality only. The one reason given for enum_cmp is the substitution argument in message #4, and the thread shows no concrete case for it.

### 2. No measurement shows the cost clears the bar used for text_cmp

fixable · level 1b, rubric item 4 · demand, security / privileges, performance on the hot path · `demand-2`

A committer weighs the risk of enum_cmp against its benefit and finds no number for the benefit. The only user in the thread reports that workarounds exist for every function on his list. The leakproof promise, which is hard to take back, then rests on the needs of one customer.

Tom Lane says text_cmp was an exception because the performance cost of not marking it was unacceptable (message #3). The thread gives no benchmark, plan, or slowdown figure for enum predicates under RLS. Andreas Lind writes that "we eventually found workarounds for all of these issues" (message #5). Demand has lasted since at least the earlier attempt on enum_eq [P1], which supports equality but does not measure the cost for ordering operators.

**Precedents.**
- [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (Rejected, 2019-03-19); thread: [link](https://www.postgresql.org/message-id/flat/2811772.0XtDgEdalL@peanuts2)

Thread: [#3 (Tom Lane, 2026-07-01)](https://www.postgresql.org/message-id/3652798.1782945225%40sss.pgh.pa.us), [#4 (Laurenz Albe, 2026-07-20)](https://www.postgresql.org/message-id/6447036b558f6c4e1cf3d695e5682c97cc3c270c.camel%40cybertec.at), [#5 (Andreas Lind, 2026-07-22)](https://www.postgresql.org/message-id/CAMxA3rvwEgfbBqRx6UB2TOAmnLp5-SDLmCuhtvKo24i6iY%2Ba2g%40mail.gmail.com)

**Question:** What slowdown do users see with enum ordering predicates under RLS, measured on a real schema, and why is the workaround Andreas Lind used not acceptable?

Counter-check: held. Message #3 sets a performance bar for enum_cmp, and the thread has no plan, benchmark or slowdown figure. Message #5 reports that workarounds exist, so the material does not show that the bar is met.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | The patch changes proleakproof for the enum comparison functions in pg_proc.dat. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The planner can push enum comparisons below RLS quals and security barrier views, including as index conditions. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | A leakproof marking is a security promise for RLS and security barrier views, and message #1 proposes a narrower threat model. |
| backpatchability | **yes** | The change edits pg_proc.dat, so it cannot go to released branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every later change to the enum cache and catalog lookup code reached by enum_cmp must keep it leakproof, with no test. |
| performance on the hot path | **yes** | see objection demand-2: No measurement shows the cost clears the bar used for text_cmp |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Would a patch that marks only enum_eq and enum_ne leakproof meet the need reported in messages #4 and #5?
2. In what concrete query or plan does enum_cmp stand in for enum_eq, so that enum_cmp must be leakproof for equality predicates to work under RLS?
3. What slowdown does a real schema show for enum ordering predicates under RLS, and why are the workarounds mentioned in message #5 not acceptable?
4. Can the case for the enum functions be argued under the existing leakproof rule, where the restricted role runs arbitrary SQL and OOM errors count?
5. Which functions does the patch mark leakproof, by name?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane replied in message #3: he accepts enum_eq as safe and wants enum_cmp rejected. A committer who knows the enum cache and catcache code would have to sign for enum_cmp.
- **Liability:** For enum_cmp, the owner signs that no present or future error path in the enum cache, syscache lookup or pg_enum scan reveals a compared value. No test checks this, and it has no end date. For enum_eq alone the liability is small.
- **Author follow-up:** The author answered the review in message #4, set the entry back to Needs review in September 2026 and moved it to the next CommitFest in October.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: For enum_cmp and the ordering operators: a concrete case where enum_cmp is used in place of enum_eq under RLS, or a measured slowdown on a real schema with enum ordering predicates that the known workarounds do not solve. The enum_eq part needs no further evidence of demand.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is a 3.4 KB catalog change in two files, and the users are sites that combine RLS with enum columns. The burden is low for enum_eq and higher for enum_cmp, so a version limited to equality, or one with a shown need for enum_cmp, could proceed.

## Appendix

### Held up in the counter-check, not listed above

- `security-1` (fixable, level 2) Narrowed leakproof threat model would apply to every future marking. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `security-2` (fixable, level 2) enum_cmp leakproofness can break with no test to catch it. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` The proposal narrows the leakproof threat model for all functions. Refuted (not specific): Duplicate of security-1. It also says the OOM part of the enum_cmp case depends on the narrower model, but message #1 argues that the cache allocations reveal only public pg_enum contents.
- `standards-compat-1` Narrower leakproof threat model is proposed but the documented definition stays. Refuted (not specific): Duplicate of security-1. The patch commits no rationale text for extension authors to read, and [P2] is about where to place a code comment, so it does not support this point.
- `maintenance-fit-1` Leakproof enum_cmp ties a large code path to an unchecked invariant. Refuted (not specific): Duplicate of security-2. [P2] is about where to place a comment on bttextcmp(), so it does not support this point.
- `maintenance-fit-2` The proof relies on a narrower leakproof standard no one has accepted. Refuted (not specific): Duplicate of security-1. The commit message argues that allocation errors reveal nothing about the compared values, which is a claim under the existing standard and does not rely on an OOM exception.
- `already-exists-1` A superuser can already mark enum_eq leakproof per installation. Refuted (already handled): Message #3 agrees that marking enum_eq leakproof is safe, so that marking puts no cost on other sites and there is nothing to keep per installation. The failure scenario is about enum_cmp, which security-2 and demand-1 already cover.

### Review angles

- prior-attempts: 1 objection(s). Message #1 points to three earlier threads and message #4 to one more, but the material does not include their content. P1's recorded reason concerns error redaction and does not block marking enum_eq leakproof, which message #3 accepts as safe. P4 concerns documentation placement and does not apply here.
- standards-compat: 1 objection(s). The change touches no syntax, GUC or psql behaviour, and pg_dump does not emit built-in function properties. Plans change under RLS only by allowing pushdown, so query results stay the same.
- replication-ondisk: 0 objection(s). The patch only flips proleakproof on built-in enum functions in pg_proc.dat and needs a catversion bump. pg_upgrade rebuilds pg_proc from the new initdb. WAL, on-disk data, standbys, subscribers, the wire protocol and the extension ABI are unaffected. It is not backpatchable, but that is normal for a catalog change and does not count against the idea.
- security: 2 objection(s). Marking enum_eq alone looks safe from this angle; message #3 agrees.
- maintenance-fit: 2 objection(s). Hot-path cost and new APIs do not apply: the change is a catalog flag, and only RLS and security barrier queries see a plan difference.
- already-exists: 1 objection(s). One objection only. The array operators listed in message #5 are outside the patch and are not judged here.
- demand: 2 objection(s). Demand for enum_eq is real and has lasted: a field user in message #5 and an earlier attempt in [P1]. The objections apply only to the parts that go beyond equality.

### Precedents retrieved

- P1: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - used in the review; reason summarised from its thread
- P2: [Code comments: why some text-handling functions are leakproof](https://commitfest.postgresql.org/patch/3510/) (cf, Returned with feedback, 2022-03-28) - used in the review; reason summarised from its thread
- P3: [enum support in btree_gin and btree_gist](https://commitfest.postgresql.org/patch/784/) (cf, Returned with feedback, 2016-12-02) - judged not relevant; reason summarised from its thread
- P4: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - used in the review; reason summarised from its thread
- P5: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P6: [Move enum storage commentary to top of section](https://commitfest.postgresql.org/patch/3696/) (cf, Returned with feedback, 2022-07-09) - judged not relevant; reason summarised from its thread
- P7: [Improved tab completion for PostgreSQL parameters in enums](https://commitfest.postgresql.org/patch/3345/) (cf, Withdrawn, 2021-10-13) - judged not relevant; reason summarised from its thread
- P8: [Use enum type replacing #define for exec state of hashjoin and mergejoin](https://commitfest.postgresql.org/patch/6027/) (cf, Withdrawn, 2026-03-30) - judged not relevant; reason summarised from its thread
- P9: [Revert to 9.6 treatment of ALTER TYPE enumtype ADD VALUE.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=28e07270768518524291d7d7906668eb67f6b8a5) (revert, Reverted, 2017-09-27) - judged not relevant
- P10: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P11: [Reverting patch just in case a compiler treats this enum as signed.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e7f4923d3b414f9920f0d479991f6dc3a9c1b739) (revert, Reverted, 2009-05-27) - judged not relevant
- P12: [Mark function arguments of type "T *" as "const T *" where possible](https://commitfest.postgresql.org/patch/6302/) (cf, Withdrawn, 2026-01-13) - judged not relevant
- P13: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant
- P14: [Revert "Provide DLLEXPORT markers for C functions via PG_FUNCTION_INFO_V1 macro."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=33cb96ba1a84c612491fb5794674a649d1a6a4d6) (revert, Reverted, 2016-11-07) - judged not relevant
- P15: [Make relfile tombstone files conditional on WAL level](https://commitfest.postgresql.org/patch/3030/) (cf, Withdrawn, 2021-10-29) - judged not relevant
- P16: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant
- P17: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P18: [GIN fillfactor](https://commitfest.postgresql.org/patch/157/) (cf, Rejected, 2015-07-21) - judged not relevant
- P19: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant
- P20: [Mark a transaction uncommittable](https://commitfest.postgresql.org/patch/4342/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P21: [GUC to improve SECURITY INVOKER safety](https://commitfest.postgresql.org/patch/4121/) (cf, Withdrawn, 2023-02-02) - judged not relevant
- P22: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P23: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant
- P24: [Use tuple-level SIREAD locks for index-only scans](https://commitfest.postgresql.org/patch/3163/) (cf, Returned with feedback, 2021-12-02) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 168 s; cost reported by the CLI: $1.99
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 94 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:02:06.999Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
