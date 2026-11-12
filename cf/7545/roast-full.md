# Design pre-review: BUG #19752: GROUP BY on a constant and a cast to a length-limited array type crashes the server while planning

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7545](https://commitfest.postgresql.org/patch/7545/) (Needs review); authors: Matheus Alcantara (matheus.alcantara)
- Thread: [BUG #19752: GROUP BY on a constant and a cast to a length-limited array type crashes the server while planning](https://www.postgresql.org/message-id/flat/19752-928339cd23b58a33%40postgresql.org), started 2026-10-08
- Messages read: 11 of 11
- Patch prose read: 0001-Match-Const-GROUP-BY-items-by-targetlist-entry.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the rule in v3. The fix addresses a reported planner crash on Postgres 18, and a committer agreed in message #2 that wrong answers are also possible. The one surviving objection, replication-ondisk-1, is a fixable back-patch point about views already stored on 18.x. The remaining risk is whether the new rule is correct in the grouping code, which needs a committer who knows that code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 11 messages and the prose of the first patch were visible. Whether v3 handles outer references, subqueries and volatile items correctly cannot be judged without the code.

## The proposal, restated

**Problem.** Since Postgres 18, `SELECT ARRAY[]::varchar(500)[] GROUP BY CAST('true' AS boolean)` crashes the planner with SIGSEGV. substitute_grouped_columns_mutator() replaces any Const that equal()s a GROUP BY item, including the internal isExplicit flag inside ArrayCoerceExpr.elemexpr, so constant folding later evaluates a Var with no tuple slot. Tom Lane notes that non-crash cases may give wrong answers.

**Proposed design.** parseCheckAggregates() records which GROUP BY items contain no Vars of the current query level. For those items, only the targetlist entry that GROUP BY identified, matched by ressortgroupref, is replaced as a whole, and the items are skipped in the equal() loop at every sublevel. The early return for Const and Param moves back to the top of the mutator, as before f5050f795ae.

**What the author claims.**
- The reported query no longer crashes.
- `select 1 as one group by rollup(one) order by one nulls first` still returns a null row, then 1.
- `select 1 as one, 1+2 as three group by rollup(one)` leaves three as 3.
- In `select 1+2 as three, (1+2)*10 as x group by rollup(three)`, the 1+2 inside x is no longer replaced by a nullable grouped Var.
- Postgres 17 is not affected; the bug comes from parse_agg.c changes in 18, not from clauses.c.

**Thread state.** Matheus Alcantara reviewed the first patch and wrote v2, which follows Tom Lane's "none except the identified TLE" rule; Andrey Rachitskiy adopted it as v3 with a test fix after a cfbot failure. Tom Lane has not commented on any patch, despite a direct request.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-08; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Views already stored on 18.x keep the bad substituted tree after the fix

fixable · level 2, rubric item 7 · catalog changes, backpatchability · `replication-ondisk-1`

A user on 18.4 creates a view whose query groups by a constant, such as `group by true` next to `ARRAY[]::varchar(500)[]`. pg_rewrite stores the parse tree with the grouped Var already substituted. After a minor update with this fix, selecting from the view still crashes or still gives the wrong result, because the stored rule is never parsed again.

The fix changes parse analysis in parse_agg.c, so it only affects trees built after the update. CREATE VIEW runs parse analysis but not planning, so the crashing tree from message #1 can already sit in pg_rewrite, as can the wrong-result trees Tom Lane mentions in message #2. pg_upgrade and dump/restore deparse and parse views again, so they repair them, but an in-place minor update does not. The thread does not say whether the fix targets 18, and it gives no note for users on what to recreate.

Thread: [#1 (PG Bug reporting form, 2026-10-08)](https://www.postgresql.org/message-id/19752-928339cd23b58a33%40postgresql.org), [#2 (Tom Lane, 2026-10-08)](https://www.postgresql.org/message-id/2530849.1791487908%40sss.pgh.pa.us), [#8 ("Matheus Alcantara", 2026-10-23)](https://www.postgresql.org/message-id/DMCBT3XQBZFK.2AR8PB94AGT40%40gmail.com)

**Question:** For back-patched 18 releases, how will views and rules stored with substituted Var-free GROUP BY items be found and repaired, and will the release notes tell users to recreate them?

Counter-check: held. CREATE VIEW stores the analyzed query without planning it, so a view created on 18.x keeps the substituted Var, and a fix in parse_agg.c does not reach it. The thread does not mention stored rules or a release note.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | Views created on 18.x keep the substituted parse tree in pg_rewrite; the patch adds no catalog change itself. |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch changes which subexpressions parse_agg.c turns into grouped Vars, so the planner receives different trees. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The fix removes a backend crash that any user can trigger without table access. |
| backpatchability | **yes** | The bug exists in released Postgres 18, and the thread does not say whether the fix goes back to 18. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a second matching rule for Var-free items to the grouping code in parse_agg.c. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is v3 meant for back-patching to 18, and if so, which query results on 18.x change after the minor update?
2. How will views and rules stored on 18.x with a substituted Var-free GROUP BY item be found, and will the release notes tell users to recreate them?
3. Does skipping Var-free items at every sublevel change results when the GROUP BY item is an outer reference or sits in a correlated subquery?
4. Do grouping sets queries with volatile Var-free items such as now() return the same results before and after v3?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane proposed the rule in the thread but has not commented on any patch version; the owner must know the RTE_GROUP substitution code in parse_agg.c.
- **Liability:** The owner signs for query results of grouping by Var-free items on 18 and later, including any result change a minor release brings to existing queries.
- **Author follow-up:** Two authors revised the patch through v3, and one fixed a test after a cfbot failure.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The trigger is narrow, but the result is a backend crash in a released branch, with possible wrong answers per message #2. The first patch is 4 KiB in parse_agg.c plus tests; v2 and v3 were not read.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Var-free grouping items stop being nulled in HAVING and other expressions. Refuted (no failure scenario): The changed HAVING result is what the rule in message #4 asks for: a Var-free GROUP BY item identifies its targetlist entry "and nothing else". ORDER BY shares that entry through ressortgroupref, and the rollup test in the author claims still returns the null row.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P5] fixed GROUP BY ALL operator semantics in transformGroupClauseExpr(), and its withdrawal was tied to GROUP BY ALL being reverted for v19; this crash fix in substitute_grouped_columns_mutator() shares only the area. The thread cites earlier work only as commits 247dea89f and f5050f795ae (message #1, message #3), which are the source of the bug and not rejected attempts.
- standards-compat: 1 objection(s). The patch adds no syntax, GUC, catalog entry or dump-visible state. The only contract concern from this angle is the change in result semantics for Var-free grouping items.
- replication-ondisk: 1 objection(s). The patch adds no WAL records, on-disk format change, protocol change or new catalog column. The only persistent state it touches is the parse trees already stored in pg_rewrite.
- security: 0 objection(s). The fix changes only which GROUP BY items parse_agg.c turns into grouped Vars; it adds no privilege, role, function, GUC, or new exposure. The current bug lets any user who can run a query crash a backend with no table access, so the patch removes an unprivileged denial of service and adds none.
- maintenance-fit: 0 objection(s). From the maintenance and fit angle there is no objection worth sending. This is a fix for a planner crash in a released branch, and it belongs in core. Its cost is a parse-time Bitmapset and a ressortgroupref check, with no executor or hot-path work. The v2/v3 rule depends on what an item contains, not on its node type. That is more general than the first patch, and future expression nodes with template fields do not have to be listed. [P5] does not apply: its recorded reason concerns GROUP BY ALL and ORDER BY operator handling, which is different code. Committability is the weak point, and it is a separate matter from the verdict. No committer has reviewed v2 or v3, and Tom Lane has not replied to the direct request in message #7. Whether the change is safe to back-patch to 18 is open. That includes Var-free items such as outer references being skipped at every sublevel, and this cannot be checked without the code.
- already-exists: 0 objection(s). This is a fix for a planner crash and possible wrong results in Postgres 18. No existing feature, setting, contrib module or extension can replace a parser bug fix, so this angle has nothing to raise. P5 is about GROUP BY ALL operator semantics, and its recorded reason does not apply here.
- demand: 0 objection(s). This is a bug fix for a reported SIGSEGV in the planner on Postgres 18 (message #1), with a reproducer and a backtrace. A committer agrees that even non-crashing cases may return wrong answers (message #2), so the demand is real and nobody would ask for more evidence of it. The cost falls on the parse_agg.c code path, not on a new user-facing feature, so a demand objection does not apply.

### Precedents retrieved

- P1: [remove_useless_groupby_columns is too enthusiastic](https://commitfest.postgresql.org/patch/3766/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P2: [Rename constants in preparation of variableness](https://commitfest.postgresql.org/patch/4828/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P3: [Replace literal 0 values with the appropriate Invalid* constants](https://commitfest.postgresql.org/patch/6480/) (cf, Withdrawn, 2026-02-13) - judged not relevant; reason summarised from its thread
- P4: [Parallel grouping sets](https://commitfest.postgresql.org/patch/2147/) (cf, Returned with feedback, 2020-07-12) - judged not relevant; reason summarised from its thread
- P5: [Fix GROUP BY ALL handling of ORDER BY operator semantics](https://commitfest.postgresql.org/patch/6952/) (cf, Withdrawn, 2026-07-16) - used in the review; reason summarised from its thread
- P6: [GUC-ify WALSender's MAX_SEND_SIZE constant](https://commitfest.postgresql.org/patch/4781/) (cf, Rejected, 2024-05-14) - judged not relevant; reason summarised from its thread
- P7: [group locking](https://commitfest.postgresql.org/patch/23/) (cf, Rejected, 2015-01-19) - judged not relevant; reason summarised from its thread
- P8: [rewriteGraphTable: Fix missing RTEs in FROM clause by setting inFromCl=true](https://commitfest.postgresql.org/patch/6602/) (cf, Withdrawn, 2026-09-10) - judged not relevant; reason summarised from its thread
- P9: [Remove useless GROUP BY columns considering unique index](https://commitfest.postgresql.org/patch/4751/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P10: [JIT expression evaluation improvements](https://commitfest.postgresql.org/patch/2501/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P11: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P12: [\[PATCH\]Refactor and unify expression construction functions in makefuncs.c](https://commitfest.postgresql.org/patch/6823/) (cf, Returned with feedback, 2026-06-01) - judged not relevant
- P13: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P14: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P15: [Revert "Do execGrouping.c via expression eval machinery."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2a41507dab0f293ff241fe8ae326065998668af8) (revert, Reverted, 2018-02-16) - judged not relevant
- P16: [Mitigation for mutating locale names on Windows](https://commitfest.postgresql.org/patch/5136/) (cf, Withdrawn, 2024-08-14) - judged not relevant
- P17: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P18: [Fix standby may crash when switching-over in certain special cases](https://commitfest.postgresql.org/patch/5199/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P19: [Add an optional timeout clause to isolationtester step](https://commitfest.postgresql.org/patch/2507/) (cf, Rejected, 2020-03-07) - judged not relevant
- P20: [Patch to clean Query after rewrite-and-analyze](https://commitfest.postgresql.org/patch/2229/) (cf, Withdrawn, 2019-08-05) - judged not relevant
- P21: [Revert "Use Foreign Key relationships to infer multi-column join selectivity".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=77ba610805e7ef9ba9c9a593ea8b1ca8f98f8bcb) (revert, Reverted, 2016-06-07) - judged not relevant
- P22: [Revert "isolationtester: don't repeat the is-it-waiting query when retrying a step."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dca369320f6023b55feb49f281d394181fc57903) (revert, Reverted, 2016-02-12) - judged not relevant
- P23: [Add a cares-about-collation column to pg_proc, so that unresolved-collation errors can be thrown at parse time](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Multi-Language_Support) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P24: [Revert "Add GROUP BY ALL".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a32733d8f10f0e35f7e4ee772ddbef552296577f) (revert, Reverted, 2026-07-17) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 118 s; cost reported by the CLI: $1.27
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 52 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:07:49.914Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
