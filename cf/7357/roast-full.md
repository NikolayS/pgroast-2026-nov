# Design pre-review: postgres_fdw: Fix costing of remote sorts without remote estimates

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7357](https://commitfest.postgresql.org/patch/7357/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [postgres_fdw: Fix costing of remote sorts without remote estimates](https://www.postgresql.org/message-id/flat/DLRIUVXIEUOQ.2E2I0B7VZLHLI%40jeltef.nl), started 2026-09-29
- Messages read: 8 of 8
- Patch prose read: v3-0001-postgres_fdw-Track-run-cost-instead-of-total-cost.patch, v3-0002-postgres_fdw-Add-tests-for-costing-local-work-bel.patch, v3-0003-postgres_fdw-Separate-remote-transfer-and-local-c.patch, v3-0004-postgres_fdw-Fix-costing-of-remote-sorts-without-.patch, v3-0005-postgres_fdw-Explain-where-remote-tlist-costing-d.patch, v1-0001-postgres_fdw-Fix-costing-of-remote-sorts-without-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

One objection survived, standards-compat-1, and it is a fixable level 2 point. Message #1 lists merge joins and merge appends turning into hash plans, and the thread shows no runtime numbers for those changes. The problem is stated and a reviewer accepts it, so levels 1a and 1b pass. The remaining work is measurements and a planner committer's judgment on new default plan shapes.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claims that 0001 changes no plans and that the sort charge applies to remote work only are unchecked. Messages #4 to #8 were not shown in full, so the v3 position on aggregate costing is judged from the commit message alone.

## The proposal, restated

**Problem.** Without use_remote_estimate, postgres_fdw costs a sorted foreign path by multiplying the whole path cost by DEFAULT_FDW_SORT_MULTIPLIER (1.2). For expensive paths that return few rows, such as joins and aggregates, this surcharge exceeds the real sort cost, so a local Sort wins and the remote sort is never pushed down. Users of postgres_fdw without remote estimates get worse plans.

**Proposed design.** Every sorted foreign path is charged 0.8 times the cost of a local sort of its output, generalizing what commit ffab494a4d did in one place. Since v2, estimate_path_cost_size() builds costs in the order the work happens (remote, transfer, local), tracks run_cost instead of total_cost, and applies the sort charge to remote work only. Local quals on base relations are charged for retrieved rows only.

**What the author claims.**
- The 1.2 surcharge does not scale with rows, so pushed-down sorts lose for aggregates and joins.
- Existing regression plans change for the better: more remote sorts, hash joins and hash aggregates instead of merge plans.
- The exact fraction does not matter, as long as the remote sort is cheaper than a local one.
- More pushdown is always better, so benchmarks are not needed.
- Plans with use_remote_estimate do not change, and the new code is simpler and fixes a startup-cost bug.

**Thread state.** Rui Zhao found LIMIT and local-filter regressions; v2 fixed them by restructuring the costing. For two aggregate costing issues Rui raised later, v3 adds comments instead of fixes, saying create_agg_path() costs the same way; nobody has replied to v3.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-09-29; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Plan changes for existing postgres_fdw users, with no runtime evidence

fixable · level 2, rubric item 8 · planner behaviour change · `standards-compat-1`

A site queries a large remote table without use_remote_estimate, and the remote side has an index on the join key. Today it gets a merge join over remotely ordered scans. After upgrade the sort charge grows as N log N, the plan flips to a hash join that builds a large local hash table, and the site finds out when query latency changes.

The regression output already shows merge joins and merge appends turning into hash plans (message #1), so the patch removes remote ordering in some places. The author's "more pushdown is always better, no need for benchmarks" (message #3) does not cover those flips. Message #2 asked for TPC-H-style runs and the remote-index case, and nobody has answered with numbers. The only way back is use_remote_estimate, which costs an extra remote round trip per path. The 0.8 fraction is fixed, while fdw_startup_cost and fdw_tuple_cost are server options.

Thread: [#1 ("Jelte Fennema-Nio", 2026-09-29)](https://www.postgresql.org/message-id/DLRIUVXIEUOQ.2E2I0B7VZLHLI%40jeltef.nl), [#2 (Narayanan Venkateswaran, 2026-10-01)](https://www.postgresql.org/message-id/CAFjuD9dj8tqqyz1NMOFU0Kr190sWz8qNbVAQApWD9p7YY9A4Wg%40mail.gmail.com), [#3 (Jelte Fennema-Nio, 2026-10-01)](https://www.postgresql.org/message-id/CAGECzQSxLNtEuWWrU1K%3D2CAiGMRSV%2BUetK_nFz7%3DpdDKnHhUdw%40mail.gmail.com)

**Question:** On a large remote table with an index on the sort key, how do runtimes compare when the plan changes from a remote-sorted merge plan to a hash plan?

Counter-check: held. Message #1 itself lists merge joins and merge appends becoming hash plans, and "more pushdown is always better" does not cover those. The thread has no runtime numbers for them, so the request for measurements stands as fixable.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | **yes** | The patch changes the cost of every sorted foreign path when use_remote_estimate is off, so existing plans change. |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The title says "Fix" and 0001 fixes a startup-cost bug, but the plan changes make the series unsuitable for stable branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch restructures estimate_path_cost_size() and adds a fixed 0.8 fraction that someone must defend later. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. On a large remote table with an index on the sort key, how do runtimes compare when a remote-sorted merge plan becomes a hash plan?
2. How often do the new plans without use_remote_estimate match the plans chosen with use_remote_estimate = true, compared with the old costing?
3. Over what range of fractions did the plans stay the same, and at what table sizes was that tested?
4. Does the series remove DEFAULT_FDW_SORT_MULTIPLIER and the 1.05 special case, or do they remain in some code path?
5. Is the startup-cost fix in 0001 meant for back-patching, separately from the patches that change plans?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows postgres_fdw path costing and the planner's sort and aggregate cost model would have to own it. No committer has posted in the visible thread.
- **Liability:** The committer signs for plan changes for every postgres_fdw user without use_remote_estimate, and for the 0.8 fraction, with reports likely after the next major release.
- **Author follow-up:** The author posted v2 and v3 in response to review within about two weeks, and credited the reviewer's findings in the commit messages.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The change affects postgres_fdw users on the default use_remote_estimate = false and adds no user-visible surface. Five patches in one contrib module are moderate to review, but the plan changes need runtime numbers and a planner-aware committer.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Plan changes for every non-estimate user, no runtime evidence or way back. Refuted (not specific): Duplicate of standards-compat-1. That one is more accurate, because it names use_remote_estimate as the existing way back, while this one says no option exists.
- `maintenance-fit-2` Fdw costing now copies create_agg_path() inaccuracies on purpose. Refuted (already handled): Message #8 says both inaccuracies exist before this patch set, and v3-0005 only adds comments. The patch writes the rule down and does not create it, and the failure depends on a hypothetical future core change.
- `demand-1` No field report or runtime numbers show the old plans hurt users. Refuted (already handled): Message #1 shows a concrete defect that commit ffab494a4d also hit, where sorted aggregate and join paths lose to a local Sort; the reviewer in message #5 agrees. The remaining request for runtime numbers duplicates standards-compat-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P9] reverted cost-based ordering of GROUP BY keys because comparator cost and group-size statistics were unreliable; this proposal does not depend on those inputs, so that reason does not apply here. Message #1 names the earlier discussion behind commit ffab494a4d, which left the 1.05 multiplier as an arbitrary choice and the question open; the proposal generalizes that commit's approach rather than repeating a rejected idea.
- standards-compat: 1 objection(s). The proposal adds no syntax, GUC, catalog object or dump state, so pg_dump, psql and SQL-standard concerns do not apply. [P9] is about planner costing, but its recorded reason was untrustworthy sort-comparison statistics, so it does not apply here.
- replication-ondisk: 0 objection(s). The proposal changes only planner costing inside contrib/postgres_fdw. Nothing in it is persistent or crosses a node boundary in a new way: no on-disk format, WAL, catalog, wire protocol or user-visible option change. The remote SQL it emits stays the same kind of query, and the postgres_fdw.h changes are internal to the module, not an extension API that others build on. Plan changes for existing users belong to the planner angle.
- security: 0 objection(s). The patch changes only planner cost estimates inside postgres_fdw when use_remote_estimate is off. It adds no privilege, GUC, catalog object or new data exposure, and it does not change which expressions are shipped to the remote server or who may run them. Plan choice alone does not open a security or privilege path, so this angle has no objection.
- maintenance-fit: 2 objection(s). Hot-path cost is limited to planning foreign paths, and the patch set is split into steps that can be committed separately, so neither gives a reason to object. No committer has posted in the visible thread. That affects committability, not the verdict.
- already-exists: 0 objection(s). The only existing alternative is use_remote_estimate, which costs extra remote EXPLAIN round trips at plan time; the patch fixes the default heuristic, adds no user-visible surface and replaces DEFAULT_FDW_SORT_MULTIPLIER with the method commit ffab494a4d already used. Neither use_remote_estimate nor any extension or hook offers that, so this angle has nothing to raise.
- demand: 1 objection(s). The problem is plausible and concerns the default configuration of a widely used module, and an earlier commit already worked around it in one place. The objection asks for evidence and does not argue that nobody needs the fix.

### Precedents retrieved

- P1: [Asynchronous execution on postgres-fdw](https://commitfest.postgresql.org/patch/291/) (cf, Rejected, 2015-08-10) - judged not relevant; reason summarised from its thread
- P2: [Push down more UPDATEs/DELETEs in postgres_fdw](https://commitfest.postgresql.org/patch/791/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P3: [postgres_fdw: evaluate placeholdervars on remote server](https://commitfest.postgresql.org/patch/1055/) (cf, Returned with feedback, 2017-09-27) - judged not relevant; reason summarised from its thread
- P4: [Async execution of postgres_fdw.](https://commitfest.postgresql.org/patch/32/) (cf, Rejected, 2015-05-09) - judged not relevant; reason summarised from its thread
- P5: [Kerberos delegation support in libpq and postgres_fdw](https://commitfest.postgresql.org/patch/3265/) (cf, Returned with feedback, 2021-11-26) - judged not relevant; reason summarised from its thread
- P6: [postgres_fdw: support parameterized foreign joins](https://commitfest.postgresql.org/patch/1042/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P7: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P8: [postgres_fdw super user checks](https://commitfest.postgresql.org/patch/824/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P9: [Revert "Optimize order of GROUP BY keys".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f4c7c410ee4a7baa06f51ebb8d5333c169691dd3) (revert, Reverted, 2022-10-03) - used in the review
- P10: [Revert "postgres_fdw: Inherit the local transaction's access/deferrable modes."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7d4667c6203d0174903a53de9d1730fbec26576b) (revert, Reverted, 2025-06-08) - judged not relevant
- P11: [Refactoring postgres_fdw/connection.c](https://commitfest.postgresql.org/patch/3782/) (cf, Withdrawn, 2023-05-23) - judged not relevant; reason summarised from its thread
- P12: [ICU as default collation provider](https://commitfest.postgresql.org/patch/1543/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P13: [Better estimate for filter cost](https://commitfest.postgresql.org/patch/1296/) (cf, Returned with feedback, 2018-10-02) - judged not relevant
- P14: [SIMD optimization for list_sort and tuple-sort](https://commitfest.postgresql.org/patch/5607/) (cf, Withdrawn, 2025-08-06) - judged not relevant
- P15: [Improve planner cost estimations for alternative subplans](https://commitfest.postgresql.org/patch/2582/) (cf, Withdrawn, 2020-11-13) - judged not relevant
- P16: [Add estimated hit ratio to Memoize in EXPLAIN to explain cost adjustment](https://commitfest.postgresql.org/patch/4236/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P17: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/167/) (cf, Returned with feedback, 2015-12-22) - judged not relevant
- P18: [Expand applicability of aggregate's sortop optimization](https://commitfest.postgresql.org/patch/4980/) (cf, Returned with feedback, 2025-07-08) - judged not relevant
- P19: [Consider incremental sort for fractional paths in generate_orderappend_paths](https://commitfest.postgresql.org/patch/3513/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P20: [cost_sort improvement](https://commitfest.postgresql.org/patch/1706/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P21: [Join pushdown support for foreign tables](https://commitfest.postgresql.org/patch/20/) (cf, Rejected, 2015-01-16) - judged not relevant
- P22: [Optimize SnapBuild by maintaining committed.xip in sorted order](https://commitfest.postgresql.org/patch/6187/) (cf, Withdrawn, 2026-04-29) - judged not relevant
- P23: [Fix unique join costings](https://commitfest.postgresql.org/patch/2050/) (cf, Returned with feedback, 2020-12-03) - judged not relevant
- P24: [Confusing docs about GetForeignUpperPaths in fdwhandler.sgml](https://commitfest.postgresql.org/patch/763/) (cf, Rejected, 2016-11-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 142 s; cost reported by the CLI: $1.81
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 63 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:44:19.466Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
