# Design pre-review: Proposal: QUALIFY clause

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6954](https://commitfest.postgresql.org/patch/6954/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [Proposal: QUALIFY clause](https://www.postgresql.org/message-id/flat/CAFY6G8euMWRjTEdRV9M%3DRL2_EJNDm5UNeHat9U1FGGwzfjGSfw%40mail.gmail.com), started 2025-07-21
- Messages read: 40 of 40
- Patch prose read: v4-0001-Add-QUALIFY-clause.patch, v4-0001-Add-QUALIFY-clause.patch.nocfbot, v0-0001-QUALIFY-clause.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal has support in the thread, on the condition that the standard reserves QUALIFY. The list has not chosen between the rewrite patch and the planner-based patch, and each has an open design cost: lost view deparse (standards-compat-2) or duplicated pushdown code (maintenance-fit-1). The claim of standard acceptance also has no source yet (prior-attempts-1). Code review makes sense after these are settled.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, it cannot be checked that the rewrite keeps column-privilege checks for columns used only in QUALIFY, or that the rewritten query gives the same results in all cases with DISTINCT, aggregates and set operations.

## The proposal, restated

**Problem.** Users who want to filter on a window function result must wrap the query in a subquery or CTE. The author wants a shorter, more readable form for this common case, which Snowflake, BigQuery, DuckDB and others already offer.

**Proposed design.** The main patch adds QUALIFY after the WINDOW clause and, after parse analysis, rewrites the query into an inner subquery and an outer query whose WHERE is the QUALIFY condition. The planner then applies existing subquery optimizations such as run conditions and qual pushdown. A second, work-in-progress patch keeps QUALIFY in the planner and covers only conditions that reference window functions.

**User-visible surface.**
- New SELECT clause `QUALIFY condition`, placed after WINDOW and before UNION/ORDER BY/LIMIT.
- QUALIFY becomes a fully reserved keyword.
- The QUALIFY condition can reference select-list output names; on a name clash the input column wins.
- With the rewrite patch, pg_get_viewdef() shows a view defined with QUALIFY in subquery form.
- EXPLAIN plans for QUALIFY queries match the equivalent hand-written subquery.

**What the author claims.**
- The SQL standard committee has accepted the QUALIFY proposal, listed as feature O001.
- The rewrite patch gets run conditions and qual pushdown with no new planner code.
- The rewrite patch implements the transformation the standard uses to define QUALIFY.
- The rewrite patch is complete for window and non-window conditions, with docs and tests.
- The author leans toward the planner-based approach for clean view deparse, but it duplicates pushdown logic.

**Thread state.** Tom Lane and others objected in 2025 to a new reserved word before standardization; Vik Fearing and David Rowley asked for a standard-style rewrite with full pushdown. Since the June 2026 revival only the author has posted, so his question on which design to pursue has no answer yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-07-21; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Rewrite design stores views in subquery form, against the reviewer's request

fixable · level 2, rubric item 13 · pg_dump / restore round-trip, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A user creates a view with QUALIFY, then reads it back with \d+ or pg_dump and gets a nested subquery with generated column names for the projected window functions. If the project later moves to the planner design, the same view deparses differently across major versions. Tools that compare schema dumps then report false differences.

In message #30 Vik Fearing asked that ruleutils print QUALIFY as written, and that any transformation happen in the rewriter. The complete patch rewrites the query during parse analysis instead, and message #38 names the deparse change as its main downside. Choosing the rewrite design fixes the stored form of every QUALIFY view, so a later change of design also changes pg_dump output for views users already have. Neither patch says whether a dumped and restored view keeps its column names and its alias-versus-input-column resolution.

Thread: [#30 (Vik Fearing, 2025-07-22)](https://www.postgresql.org/message-id/c5e9fe6e-0f93-4933-97f1-85a0acc774be%40postgresfriends.org), [#34 (Vik Fearing, 2025-07-25)](https://www.postgresql.org/message-id/b4ed9908-f000-4a5d-aae8-625607f80757%40postgresfriends.org), [#38 ("Matheus Alcantara", 2026-06-29)](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com)

**Question:** Does a QUALIFY view dumped by pg_dump restore with the same column names and meaning, and which stored form will views keep if the project later switches to the planner design?

Counter-check: held. Message #30 asked that ruleutils print QUALIFY as written, and message #38 names the lost deparse as the main downside of the rewrite patch. Nobody has answered the design question, and neither patch states the dump and restore result.

### 2. Planner-based design copies window pushdown logic into a second place

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), planner behaviour change · `maintenance-fit-1`

A later patch improves run conditions or qual pushdown below WindowAgg in set_subquery_pathlist() and does not update the QUALIFY path. Hand-written subqueries then get the new plan and QUALIFY queries do not. Nothing fails, so users find out only when they compare EXPLAIN output.

The author leans toward the planner-based patch (message #38) and says it duplicates the run-condition and qual-pushdown code. David Rowley (message #36) said acceptance depends on how much code is reused, and he named a past optimization that would have needed a second copy. Richard Guo (message #37) asked that QUALIFY inherit every existing window optimization. The refactoring of check_and_push_window_quals() into shared helpers is proposed but has not been done.

Thread: [#36 (David Rowley, 2025-07-29)](https://www.postgresql.org/message-id/CAApHDvq5UTReLSdwEL9WnmDEW1VDg-EYTmszPVOGwf8qH-cj9Q%40mail.gmail.com), [#37 (Richard Guo, 2025-07-29)](https://www.postgresql.org/message-id/CAMbWs48vmkAHpdPoX9_gnbcVMv8c2w2Y6UAFzZjTynz9fVGsuA%40mail.gmail.com), [#38 ("Matheus Alcantara", 2026-06-29)](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com)

**Question:** Can the subquery pushdown helpers be refactored first, as a separate patch, so that QUALIFY and subqueries share a single code path?

Counter-check: held. It applies only to the planner-based patch, which the author prefers. Message #38 confirms the duplication, messages #36 and #37 make reuse a condition, and the shared helpers are not written.

### 3. Standardization condition from 2025 is claimed but not yet shown

fixable · level 1b, rubric item 6 · prior rejection, SQL-standard conflict or non-standard syntax, backpatchability · `prior-attempts-1`

An application has a table or column named qualify. After an upgrade to a release with this patch, its unquoted queries fail with syntax errors. If the standard later changes the keyword or its scope, the project cannot cite the standard to explain the break.

In messages #4 and #6, Tom Lane accepted the reserved word only once QUALIFY is in the standard, so the breakage can be blamed on SQL:20xx. Vik Fearing agreed in messages #11 and #26 to hold back until then. Message #38 says the proposal is accepted as feature O001 but gives no reference, and the conformance doc hunk is truncated. No reviewer has confirmed the claim since the revival, so the condition from 2025 is still open.

Thread: [#4 (Tom Lane, 2025-07-21)](https://www.postgresql.org/message-id/440577.1753108882%40sss.pgh.pa.us), [#6 (Tom Lane, 2025-07-21)](https://www.postgresql.org/message-id/463108.1753119052%40sss.pgh.pa.us), [#9 (Vik Fearing, 2025-07-21)](https://www.postgresql.org/message-id/88d8f6c9-d19d-4cee-bbc5-826987c14587%40postgresfriends.org), [#11 (Vik Fearing, 2025-07-21)](https://www.postgresql.org/message-id/0b190baa-460d-48a8-9f7f-ca8c8cfe13bd%40postgresfriends.org), [#26 (Vik Fearing, 2025-07-22)](https://www.postgresql.org/message-id/737a77a2-83eb-4b97-a109-e58d3ce1189a%40postgresfriends.org), [#38 ("Matheus Alcantara", 2026-06-29)](https://www.postgresql.org/message-id/DJLM1EHPGO3K.3PDCCHL8OZ08V%40gmail.com)

**Question:** Which standard document or committee decision records QUALIFY as feature O001, with the keyword reserved and the semantics the patch implements?

Counter-check: held. Messages #4, #6 and #11 made standard acceptance the condition for the reserved word. Message #38 states acceptance without a reference, the conformance hunk is truncated, and no reviewer has confirmed it since.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | Both patches change parsenodes.h, and the planner-based patch also changes optimizer/paths.h. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds a QUALIFY clause and makes QUALIFY a fully reserved keyword. |
| new GUC | no |  |
| planner behaviour change | **yes** | The planner-based patch adds QUALIFY pushdown and run conditions in allpaths.c and planner.c. |
| pg_dump / restore round-trip | **yes** | With the rewrite patch, a view defined with QUALIFY deparses and dumps in subquery form. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The new reserved word breaks unquoted uses of qualify as an identifier on upgrade. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The planner-based patch keeps a second copy of the window pushdown logic that later patches must update. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which standard document or committee decision records QUALIFY as feature O001, with the keyword reserved and the semantics the patch implements?
2. Does a view defined with QUALIFY dump and restore with the same column names and the same alias-versus-input-column resolution under the rewrite patch?
3. Can the subquery pushdown helpers be refactored first, as a separate patch, so that QUALIFY and subqueries share one code path?
4. How does the planner-based patch filter QUALIFY conditions that contain no window function?
5. Is the crash David Rowley reported with an alias in QUALIFY fixed in v4, and does a regression test cover it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the grammar and keyword policy, and one who knows window function planning. Committers commented earlier in the thread, but none has posted since the June 2026 revival.
- **Liability:** The owner signs for a permanent reserved keyword, the stored and dumped form of QUALIFY views, and, with the planner-based patch, a second pushdown path to keep in step.
- **Author follow-up:** The author revived the thread in June 2026, posted v4 with two alternative patches, and kept the entry rebased across three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Several other systems ship QUALIFY and users migrating from them gain, but the choice between two patches of about 86 KiB each is open. Reviewing either in detail before that choice risks wasted work.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Fully reserved QUALIFY breaks existing identifiers, and the standard status lacks a source. Refuted (not specific): Duplicate of prior-attempts-1. The thread already accepted the reserved-word breakage once the standard reserves QUALIFY (messages #6, #11), so only the missing source remains open.
- `replication-ondisk-1` Rewrite patch stores QUALIFY views in pg_rewrite in subquery form permanently. Refuted (wrong facts): The rewrite patch stores an ordinary subquery Query tree, with no new stored format, and pg_upgrade restores views from pg_dump text. The remaining point, that the QUALIFY spelling is lost, is a duplicate of standards-compat-2.
- `maintenance-fit-2` Parse-analysis rewrite stores the subquery form in every view. Refuted (not specific): Duplicate of standards-compat-2. The claim of two stored forms does not hold, because old views would be ordinary subquery views that need no special support.
- `maintenance-fit-3` Non-window QUALIFY needs a new executor filter for one clause. Refuted (no failure scenario): The scenario is about code that neither patch contains, and the complete rewrite patch needs no new executor filter. The cost to queries with a Result node is asserted without support, and message #39 already narrows the cases.
- `already-exists-1` The subquery form already does this, so core inclusion rests on the standard. Refuted (not specific): Duplicate of prior-attempts-1. The thread weighed the subquery alternative and accepted the clause once standardized (messages #4, #11, #20, #27), so only the missing source remains.
- `demand-1` Reserved keyword cost for all, ergonomic gain only for QUALIFY users. Refuted (not specific): Duplicate of prior-attempts-1. The thread weighed the reserved-word cost against the gain and accepted it on standardization (messages #6, #11), and messages #1, #2 and #9 show demand from other systems and the committee.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent covers QUALIFY. [P7] is about ruleutils output for JOIN USING aliases, and its portability reason does not apply here. The only earlier objection is in this thread: the conditional acceptance in messages #4 and #6.
- standards-compat: 2 objection(s). Two objections raised. Placement after WINDOW and name resolution with input columns winning match what reviewers asked for in messages #10 and #15, so those points are not raised.
- replication-ondisk: 1 objection(s). Neither patch touches WAL, relation files, SLRUs, the control file or the wire protocol. The only persistent state is the stored rule for views, covered above. [P7] concerns dump portability to older servers and does not apply here.
- security: 0 objection(s). QUALIFY adds no new privilege, object type or exposure path. It filters rows the user can already read in a query the user writes. RLS, column privileges and security_barrier quals still apply to the base relations in both designs, and pushdown into security barrier views uses the existing leakproof checks. Without the code, it cannot be checked that the rewrite keeps column-privilege checks for columns used only in QUALIFY.
- maintenance-fit: 3 objection(s). Reserved-keyword breakage and the claim of standard acceptance are left to other angles. No committer has posted since the June 2026 revival, so nobody has answered the design question.
- already-exists: 1 objection(s). Only one objection from this angle. No extension can add SELECT grammar, so if the standard status is confirmed, the existing subquery idiom does not by itself argue against core placement.
- demand: 1 objection(s). Demand is partly shown: many other systems ship QUALIFY, and migrations from them would benefit. The only demand-side objection is that the reserved-word cost depends on a standard status the material does not verify.

### Precedents retrieved

- P1: [Implement <null treatment> for window functions](https://commitfest.postgresql.org/patch/2628/) (cf, Returned with feedback, 2021-03-25) - judged not relevant; reason summarised from its thread
- P2: [window function improvements WIP](https://commitfest.postgresql.org/patch/2419/) (cf, Returned with feedback, 2020-09-17) - judged not relevant; reason summarised from its thread
- P3: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant; reason summarised from its thread
- P4: [possibility to write window function in PL languages (like PLpgSQL)](https://commitfest.postgresql.org/patch/2697/) (cf, Withdrawn, 2021-01-27) - judged not relevant; reason summarised from its thread
- P5: [Consider parallel for LATERAL subqueries having LIMIT/OFFSET](https://commitfest.postgresql.org/patch/2851/) (cf, Returned with feedback, 2023-03-23) - judged not relevant; reason summarised from its thread
- P6: [\[PATCH\] Add UPDATE WHERE OFFSET IN clause](https://commitfest.postgresql.org/patch/3531/) (cf, Returned with feedback, 2022-04-08) - judged not relevant; reason summarised from its thread
- P7: [Use JOIN USING aliases in ruleutils.c](https://commitfest.postgresql.org/patch/3538/) (cf, Returned with feedback, 2022-03-25) - used in the review; reason summarised from its thread
- P8: [Parallelize correlated subqueries that execute within each worker](https://commitfest.postgresql.org/patch/3246/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P9: [Changing WAL Header to reduce contention during ReserveXLogInsertLocation()](https://commitfest.postgresql.org/patch/1455/) (cf, Returned with feedback, 2018-07-09) - judged not relevant
- P10: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P11: [Allow HAVE_UCOL_STRCOLLUTF8 flag to be set conditionally on Windows](https://commitfest.postgresql.org/patch/1188/) (cf, Rejected, 2017-08-11) - judged not relevant
- P12: [Allow FunctionScans to pipeline results](https://commitfest.postgresql.org/patch/2372/) (cf, Withdrawn, 2020-03-23) - judged not relevant
- P13: [Windows Service status is set to "SERVICE_RUNNING" when timeout is occurred (version 9.6 or earlier).](https://commitfest.postgresql.org/patch/1970/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P14: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P15: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P16: [Make deparsing of column defaults faster](https://commitfest.postgresql.org/patch/1657/) (cf, Returned with feedback, 2018-09-24) - judged not relevant
- P17: [Revert changes about warnings/errors for placeholders.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=cab5b9ab2c066ba904f13de2681872dcda31e207) (revert, Reverted, 2021-12-27) - judged not relevant
- P18: [HINT on qualified LHS references in UPDATE .. SET](https://commitfest.postgresql.org/patch/60/) (cf, Rejected, 2015-01-16) - judged not relevant
- P19: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P20: [remove C++ keywords](https://commitfest.postgresql.org/patch/1075/) (cf, Returned with feedback, 2017-04-06) - judged not relevant
- P21: [Add initdb option to initialize cluster with non-standard xid/mxid/mxoff.](https://commitfest.postgresql.org/patch/3634/) (cf, Rejected, 2023-03-21) - judged not relevant
- P22: [Patch to clean Query after rewrite-and-analyze](https://commitfest.postgresql.org/patch/2229/) (cf, Withdrawn, 2019-08-05) - judged not relevant
- P23: [Support ALTER INDEX ... WHERE ... clause](https://commitfest.postgresql.org/patch/502/) (cf, Returned with feedback, 2016-04-08) - judged not relevant
- P24: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 157 s; cost reported by the CLI: $3.19
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 93 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:10:30.640Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
