# Design pre-review: ANSI SQL proposal: SELECT DISTINCT ON (... ORDER BY ...) and UNION DISTINCT ON (... ORDER BY ...)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7307](https://commitfest.postgresql.org/patch/7307/) (Needs review); authors: Hannu Krosing (hannukrosing)
- Thread: [ANSI SQL proposal: SELECT DISTINCT ON (... ORDER BY ...) and UNION DISTINCT ON (... ORDER BY ...)](https://www.postgresql.org/message-id/flat/CAMT0RQR_QqLqKdLgh3zYk7yA7oH9RJBVF%2BkZJ607wjJD9WKH9Q%40mail.gmail.com), started 2026-08-01
- Messages read: 10 of 10
- Patch prose read: v3-0001-Refactor-set-operation-group-clauses-to-decouple-.patch, v3-0002-Support-inline-ORDER-BY-in-DISTINCT-ON-for-SELECT.patch, v3-0003-Support-UNION-DISTINCT-ON-for-non-recursive-set-o.patch, v3-0004-Support-dynamic-pruning-in-recursive-UNION-DISTIN.patch, v3-0005-Fix-bugs-security-issues-and-optimizations-in-DIS.patch, xxx000-distinct-on-clause.md (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

Level 1b fails on two fixable objections. standards-compat-1: message #9 says wider approval of the syntax comes only after v19, and message #4 endorses only the inline ORDER BY, so the UNION form would ship ahead of the committee. demand-1: the thread shows no non-recursive workload that DISTINCT ON over a UNION ALL subquery does not already serve. The inline ORDER BY for SELECT drew no surviving objection.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Two level 2 objections survived but were not evaluated: the undocumented recursive limits (standards-compat-2) and the series shape (maintenance-fit-2). The security_barrier pushdown rule and the full set of planner paths that relied on the DISTINCT ON and ORDER BY match cannot be judged without the code.

## The proposal, restated

**Problem.** DISTINCT ON requires its keys to lead the query's ORDER BY, so picking the kept row and sorting the output cannot be done separately without a subquery. There is no way to deduplicate on a subset of columns during a UNION. In WITH RECURSIVE, UNION ALL with CYCLE explores every path, so shortest-path queries grow combinatorially.

**Proposed design.** DISTINCT ON takes an optional ORDER BY inside its parentheses that picks the row kept per key, separate from the outer ORDER BY. UNION gains DISTINCT ON (keys [ORDER BY ...]), planned as HashAggregate or Sort plus Unique. In WITH RECURSIVE, a better row replaces the stored entry and goes back into the work table, so the recursion keeps the best row per key until convergence.

**User-visible surface.**
- New syntax: select distinct on (keys order by sort_keys) ..., with the outer ORDER BY no longer required to match the keys.
- New syntax: select ... union distinct on (keys [order by sort_keys]) select ..., also for nested and multi-way set operations.
- WITH RECURSIVE with UNION DISTINCT ON (... ORDER BY ...) prunes rows per key; SEARCH and CYCLE are rejected with an inline ORDER BY.
- EXPLAIN shows a "Sort Key:" line for HashAggregate and RecursiveUnion when tie-break sort columns are used.
- pg_get_viewdef() deparses both new forms.

**What the author claims.**
- The inline ORDER BY separates the choice of kept row from the output order, so no subquery is needed.
- Recursive UNION DISTINCT ON allows shortest-path and Bellman-Ford style search in plain SQL without infinite loops.
- The syntax follows the SQL precedent of ORDER BY inside OVER, WITHIN GROUP and aggregate calls.
- The author drafted a SQL standard discussion paper and plans to seek wider approval of the syntax after the v19 release.
- The author says the design looks similar to DuckDB's USING KEY.

**Thread state.** Vik Fearing supports the inline ORDER BY but asked for a UNION DISTINCT ON use case, which Hannu gave as a recursive shortest-path query. Manu reported no wrong results in about 42,000 cases, an 820x recursive speedup and a planner fix for group estimates, which v3 does not include yet.

## 2. Prior attempts

### Removing useless DISTINCT clauses (related)

- **Link:** [https://commitfest.postgresql.org/patch/1369/](https://commitfest.postgresql.org/patch/1369/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAKJS1f8UMJ137sRuVSnEMDDpa57Q71JuLZi4yLCFMekNYVYqaQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAKJS1f8UMJ137sRuVSnEMDDpa57Q71JuLZi4yLCFMekNYVYqaQ@mail.gmail.com)
- **Date:** decided 2018-04-07; started 2017-11-06
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane found that removing distinctClause items breaks the parser-enforced match between DISTINCT and ORDER BY, which create_distinct_paths relies on. With a primary key on (a,b), `select distinct * from t1 order by a,c,b` hits an assertion and crashes the server. He judged the fix too large for the last day of the commitfest.
- **Reason (quote):** "Once I noticed that, I became convinced that this patch actually breaks planning of sort/unique-based DISTINCT: if we have a pkey a,b, but the ORDER BY list is a,c,b, then we will sort by a,c,b but the Unique node will be told it only needs to unique-ify on the first two sort columns." ([message](https://www.postgresql.org/message-id/4779.1523134536%40sss.pgh.pa.us))
- **What is different this time:** P2 removed DISTINCT columns inside the planner; this proposal lets the user separate DISTINCT ON keys from the outer ORDER BY. Both relax the parser-enforced match between DISTINCT and ORDER BY.


## 3. Objections a reviewer is likely to raise

### 1. New DISTINCT ON spellings ship before the standard committee has reviewed them

fixable · level 1b, rubric item 6 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

Postgres ships `union distinct on (k order by c)` in a release, and users put it in views. The committee later adopts a different spelling or different tie-break rules for subset deduplication. Postgres must then support both forms indefinitely, and portable SQL using the standard form behaves differently from existing Postgres views.

The discussion paper (message #1) is a draft, and message #9 says wider approval will be sought only after v19. The standard already covers per-key selection with QUALIFY and FETCH FIRST ... PARTITIONS BY (message #4), and DuckDB uses USING KEY for the recursive case (message #6), so the committee has other spellings to choose from. Vik Fearing endorsed the inline ORDER BY for SELECT but questioned UNION DISTINCT ON, and that is the part with no precedent at all.

Thread: [#1 (Hannu Krosing, 2026-08-01)](https://www.postgresql.org/message-id/CAMT0RQR_QqLqKdLgh3zYk7yA7oH9RJBVF%2BkZJ607wjJD9WKH9Q%40mail.gmail.com), [#4 (Vik Fearing, 2026-09-17)](https://www.postgresql.org/message-id/e9b65da3-9326-4951-91d2-95f1e4c1b2a9%40postgresfriends.org), [#6 (Hannu Krosing, 2026-09-18)](https://www.postgresql.org/message-id/CAMT0RQRjvzGYNgUBzmRaRuX64m-VM2n6Stm3ANmzigoSb_Wf1w%40mail.gmail.com), [#9 (Hannu Krosing, 2026-10-25)](https://www.postgresql.org/message-id/CAMT0RQShho4N09N1BnPRjPts7SGxiM5pH8SK5tGBC%3D4wAoxkDQ%40mail.gmail.com)

**Question:** Has WG 3 seen the UNION DISTINCT ON form, and would the Postgres design still hold if the committee chose USING KEY or different semantics?

Counter-check: held. Message #9 confirms that wider approval of the syntax comes only after v19, and message #4 endorses only the inline ORDER BY. Postgres already carries non-standard DISTINCT ON, but that does not answer the risk for the new UNION form.

### 2. Non-recursive UNION DISTINCT ON has no use case beyond a workaround

fixable · level 1b, rubric item 3 · demand, SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years"), doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A user who wants the newest row per key across two tables writes DISTINCT ON over a UNION ALL subquery today, and it works. With the patch the project carries new set-operation grammar, Append-flattening guards and pushdown checks for that case. No user story shows the subquery form failing.

In message #4 Vik Fearing asked for a UNION DISTINCT ON use case, and message #5 gave only a recursive shortest-path query. In message #8 the non-recursive form was 1.36x slower than the classic rewrite at 200k groups. Message #10 shows a gain only with an unmerged estimate diff, and a loss again at 500k groups. The discussion paper argues for early pruning but gives no workload where the rewrite is too slow.

Thread: [#4 (Vik Fearing, 2026-09-17)](https://www.postgresql.org/message-id/e9b65da3-9326-4951-91d2-95f1e4c1b2a9%40postgresfriends.org), [#5 (Hannu Krosing, 2026-09-18)](https://www.postgresql.org/message-id/CAMT0RQRCxNHoOjLUnFsZk8aNQqFDkYs3n2P5HecKYigAGJk0xQ%40mail.gmail.com), [#8 (Manu, 2026-10-17)](https://www.postgresql.org/message-id/179226324434.751863.15815974755418841666%40gmail.com), [#10 (Manu, 2026-10-26)](https://www.postgresql.org/message-id/179302661466.1417973.8633344727673033675%40gmail.com)

**Question:** Which real non-recursive queries need UNION DISTINCT ON, and why is DISTINCT ON over a UNION ALL subquery not enough for them?

Counter-check: held. The discussion paper gives a non-recursive example (active and archived users), but DISTINCT ON over a UNION ALL subquery does that job today. Messages #8 and #10 show no consistent gain over that rewrite, and the thread names no workload that needs the new form.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The series adds fields to parse, path and plan nodes and a new ReplaceTupleHashEntryIfBetter() in executor.h. |
| SQL-standard conflict or non-standard syntax | **yes** | It adds DISTINCT ON (keys ORDER BY ...) and UNION DISTINCT ON, both non-standard, with only a draft discussion paper behind them. |
| new GUC | no |  |
| planner behaviour change | **yes** | It adds HashAggregate and MergeAppend plus Unique paths for DISTINCT ON and changes set-operation grouping and qual pushdown. |
| pg_dump / restore round-trip | **yes** | Views can contain both new forms, so dump and restore depend on the new pg_get_viewdef() deparsing in ruleutils.c. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | see objection standards-compat-1: New DISTINCT ON spellings ship before the standard committee has reviewed them |
| maintenance burden ("who maintains this in 10 years") | **yes** | RecursiveUnion gains a replace-and-requeue mode, and about 30 files change across parser, planner, executor and rewriter. |
| performance on the hot path | **yes** | The series changes execGrouping.c and nodeAgg.c, which every hashed aggregate uses; message #8 reports no measurable regression. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Non-recursive UNION DISTINCT ON can be written today as DISTINCT ON over a UNION ALL subquery. |

## 5. Questions the author must answer before anyone reviews code

1. Has WG 3 seen the UNION DISTINCT ON form, and would the Postgres design still hold if the committee chose USING KEY or different semantics?
2. Which real non-recursive queries need UNION DISTINCT ON, and why is DISTINCT ON over a UNION ALL subquery not enough for them?
3. Can the series be reordered so that 0001 plus the inline ORDER BY patch stand alone, each correct without 0005?
4. Will the docs define the recursive mode as replacement, state the negative-cycle case, and explain why SEARCH and CYCLE are rejected with an inline ORDER BY?
5. Does the result of recursive pruning depend on the order in which rows arrive, and if so, in which cases?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows set-operation planning, hashed aggregation and nodeRecursiveunion.c. Only Vik Fearing, on syntax, and Manu, on testing, have replied.
- **Liability:** The owner signs for two new syntax forms that cannot be removed once views use them, and for a second RecursiveUnion mode that every later WITH RECURSIVE change must preserve.
- **Author follow-up:** The author posted a rebase and a v3 with a fix patch in message #7, and answered the use-case question in message #5.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** a sizeable minority
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** The inline ORDER BY answers a common pattern and has support in message #4, but the series ties it to UNION DISTINCT ON and the recursive mode. Those two parts carry the open syntax and demand questions, and the series spans about 30 files.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-2` (fixable, level 2) Recursive UNION DISTINCT ON departs from WITH RECURSIVE semantics without documenting it. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) No committer engaged; the five-patch series is all-or-nothing. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-1` Relaxing the DISTINCT ON and ORDER BY match drops a planner invariant. Refuted (no failure scenario): The objection names no path that still assumes the match. 0005 fixes the needed_pathkeys prefix check, and message #8 reports 0 divergences in about 42,000 cases on the full series, including the MergeAppend plus Unique paths.
- `prior-attempts-2` Recursive UNION DISTINCT ON never terminates on negative cycles. Refuted (not specific): Duplicate of standards-compat-2, which covers the negative-cycle case together with the SEARCH and CYCLE restriction. [P4] concerns a parse-tree rewrite and must not be used as support.
- `maintenance-fit-1` Recursive pruning adds a second RecursiveUnion mode with open termination. Refuted (not specific): Duplicate of standards-compat-2 on termination and on SEARCH and CYCLE. The remaining part, that a later developer forgets the new mode, names no concrete path and would apply to any new executor mode.
- `maintenance-fit-3` UNION DISTINCT ON fixes non-standard syntax before the committee sees it. Refuted (not specific): Duplicate of standards-compat-1, which states the same risk with the same thread references and a more concrete failure scenario.
- `demand-2` Recursive pruning demand rests on one puzzle and one paper. Refuted (already handled): Message #8 shows the existing SQL form cannot do the job: UNION ALL with CYCLE no longer finishes within 2 GB of temp space from a 6x6 grid up. DuckDB's USING KEY, cited in message #6, shows another system added this capability.

### Review angles

- prior-attempts: 2 objection(s). No listed precedent proposed the same idea. The two related rows match risks this patch takes on, so both are raised above.
- standards-compat: 2 objection(s). Two objections raised. No backward-compatibility break was found for existing DISTINCT ON queries. New syntax failing to load into an older server is normal for any new syntax and is not counted as an objection.
- replication-ondisk: 0 objection(s). The proposal changes no on-disk format, WAL record, replication path or wire protocol message. New parse-node and plan-node fields land in stored view rules and in plans sent to parallel workers, but a major release with a catversion bump already covers that. pg_upgrade rebuilds views from SQL, and the patch adds pg_get_viewdef() deparsing for both new forms. Changes to executor and planner structs affect extensions only at a major release, as usual.
- security: 0 objection(s). The proposal adds query syntax and executor strategies with no new privilege or exposure, so the security angle finds no design-level objection. Any user can already write a WITH RECURSIVE query that runs until statement_timeout, so the negative-cycle loop adds no new denial of service. Comparators and user operators run as in existing DISTINCT ON sorts. The 'security issues' that 0005 fixes are memory-safety bugs in the code, which is out of scope here. The pushdown rule for security_barrier views over UNION DISTINCT ON cannot be checked without the code.
- maintenance-fit: 3 objection(s). Hot-path cost was not raised as an objection: message #8 found no measurable regression for untouched features, and the hashing cost falls on users of the new syntax.
- already-exists: 0 objection(s). Each existing alternative is either a convenience gap or cannot do the job. The inline ORDER BY replaces one subquery around classic DISTINCT ON, and Vik Fearing endorsed it in message #4. Non-recursive UNION DISTINCT ON can be written today as DISTINCT ON over a UNION ALL subquery. But it shares syntax and machinery with the recursive form, so dropping it saves little. Recursive per-key pruning has no equivalent today: UNION compares whole rows and UNION ALL with CYCLE explores every path (message #8). No extension or hook offers it outside core.
- demand: 2 objection(s). The inline ORDER BY in SELECT DISTINCT ON has support from Vik Fearing in message #4 and answers a common pattern, so no demand objection is raised against that part.

### Precedents retrieved

- P1: [parallel distinct union and aggregate support patch](https://commitfest.postgresql.org/patch/2769/) (cf, Returned with feedback, 2021-11-08) - judged not relevant; reason summarised from its thread
- P2: [Removing useless DISTINCT clauses](https://commitfest.postgresql.org/patch/1369/) (cf, Returned with feedback, 2018-04-07) - used in the review; reason summarised from its thread
- P3: [Remove useless distinct clauses](https://commitfest.postgresql.org/patch/2663/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P4: [Allow multiple recursive self-references](https://commitfest.postgresql.org/patch/3046/) (cf, Returned with feedback, 2022-08-02) - used in the review; reason summarised from its thread
- P5: [ALTER TABLE: emit NOTICE when ONLY is omitted but no partition recursion occurs](https://commitfest.postgresql.org/patch/6424/) (cf, Withdrawn, 2026-01-22) - judged not relevant; reason summarised from its thread
- P6: [Union Replacement of OR logic](https://commitfest.postgresql.org/patch/4715/) (cf, Rejected, 2024-07-24) - judged not relevant; reason summarised from its thread
- P7: [allow partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/2787/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P8: [Add support for Distinct in Window aggregates](https://commitfest.postgresql.org/patch/4243/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P9: [\[PoC\] Implementation of distinct in Window Aggregates: take two](https://commitfest.postgresql.org/patch/4519/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P10: [add explicit partial union-all and improve parallel subquery costing](https://commitfest.postgresql.org/patch/3070/) (cf, Returned with feedback, 2021-12-02) - judged not relevant
- P11: [Remove redundant DISTINCT when GROUP BY already guarantees uniqueness](https://commitfest.postgresql.org/patch/6869/) (cf, Withdrawn, 2026-07-23) - judged not relevant
- P12: [Convert join OR clauses into UNION queries](https://commitfest.postgresql.org/patch/1001/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P13: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - judged not relevant
- P14: [(ALTER SYSTEM) SET +=/-= operator support](https://commitfest.postgresql.org/patch/2750/) (cf, Returned with feedback, 2021-03-12) - judged not relevant
- P15: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P16: [NOTIFY and pg_notify performance when deduplicating notifications](https://commitfest.postgresql.org/patch/1820/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P17: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P18: [Add inline comments to the pg_hba_file_rules view](https://commitfest.postgresql.org/patch/4546/) (cf, Withdrawn, 2023-10-10) - judged not relevant
- P19: [SQL Inlining Behavior](https://commitfest.postgresql.org/patch/1965/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P20: [Corresponding by clause for set operation](https://commitfest.postgresql.org/patch/1016/) (cf, Returned with feedback, 2017-04-04) - judged not relevant
- P21: [Apply a better fix to mdunlinkfork().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=85d8b30724c0fd117a683cc72706f71b28463a05) (revert, Partially reverted, 2022-11-09) - judged not relevant
- P22: [Revert "Convert *GetDatum() and DatumGet*() macros to inline functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e8d78581bb864369035c736a82ecfb57482a1a09) (revert, Reverted, 2022-09-12) - judged not relevant
- P23: [Extend more usecase for planning time partition pruning and init partition pruning.](https://commitfest.postgresql.org/patch/2954/) (cf, Withdrawn, 2021-02-19) - judged not relevant
- P24: [Revert "Make some use of anonymous unions \[pgcrypto\]"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5aabe6d58e08179cc656539c4bd0145525bf22f) (revert, Reverted, 2025-09-30) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 12; wall time: 157 s; cost reported by the CLI: $2.69
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 97 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:39:12.475Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
