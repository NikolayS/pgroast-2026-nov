# Design pre-review: RLS bypass: ON CONFLICT DO UPDATE/SELECT evaluates WHERE before RLS check

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7403](https://commitfest.postgresql.org/patch/7403/) (Needs review); authors: Joao Detomini (joaodetomini)
- Thread: [RLS bypass: ON CONFLICT DO UPDATE/SELECT evaluates WHERE before RLS check](https://www.postgresql.org/message-id/flat/CABH8dKzYzRKD4izbushndHvvsEQg9b6m3w9ncYAGMbZ5m%2B9gNg%40mail.gmail.com), started 2026-10-06
- Messages read: 1 of 1
- Patch prose read: 0001-Check-RLS-policies-before-evaluating-the-ON-CONFLICT.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the reorder itself; the leak has a repro and the new order matches the CREATE POLICY docs. The one surviving objection, standards-compat-1, is fixable and concerns the backpatch through 14, which turns a silent no-op into an RLS error in minor releases and is untested on 14 and 15. That is a per-branch risk decision for a committer who knows RLS and the executor.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so it is unchecked whether the reorder covers every path where the WHERE clause sees the existing row, or changes locking or EvalPlanQual behavior. The DO SELECT part exists only on master, so the stable-branch patch would differ.

## The proposal, restated

**Problem.** ON CONFLICT DO UPDATE and DO SELECT evaluate the user's WHERE clause on the existing row before checking that row against RLS policies. A user who cannot see a row can read its columns through a leaky function or use a plain predicate as an oracle. They only need to hit the row's unique key.

**Proposed design.** The patch swaps the order in ExecOnConflictUpdate() and ExecOnConflictSelect() so the WCO_RLS_CONFLICT_CHECK runs before the ON CONFLICT WHERE qual. A row hidden by policy now raises the RLS error even when the WHERE clause would be false. The patch adds regression tests that use f_leak() in the WHERE clause of both paths.

**User-visible surface.**
- ON CONFLICT DO UPDATE ... WHERE and DO SELECT ... WHERE now raise the RLS error for a hidden conflicting row even when WHERE is false.

**What the author claims.**
- The current order lets hidden column values leak through a WHERE function or act as a match oracle.
- The CREATE POLICY docs say the existing row is checked against USING expressions first, so the fix matches the docs.
- The DO UPDATE ordering dates from 168d5805e4 (2015); DO SELECT (88327092ff) copied it.
- With the patch the repro no longer leaks and the regress suite passes.
- The fix should be backpatched through 14.

**Thread state.** Only the first message is visible. A reviewer added himself in the CommitFest entry and removed himself the same day.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-06; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatch turns silent no-op upserts into RLS errors in minor releases

fixable · level 2, rubric item 6 · backpatchability · `standards-compat-1`

A multi-tenant application runs `insert ... on conflict (id) do update ... where <false for other tenants>` under RLS. Today, a conflict with another tenant's hidden row is a silent no-op. After a 14.x–17.x minor update, the same statement raises the RLS error, and the application finds out in production through failed transactions.

Message #1 and the commit message state the behavior change and ask for Backpatch-through: 14. The proposal does not say whether any applications depend on a false WHERE skipping the error, and minor releases normally avoid such changes. The author tested only master and 16beta2, and DO SELECT exists only on master, so the backpatch covers just DO UPDATE on 14 and 15, untested there. A committer would likely ask whether the oracle needs this change on stable branches, or whether master-only plus a release note is enough.

Thread: [#1 (Joao Detomini, 2026-10-06)](https://www.postgresql.org/message-id/CABH8dKzYzRKD4izbushndHvvsEQg9b6m3w9ncYAGMbZ5m%2B9gNg%40mail.gmail.com)

**Question:** Which existing upsert patterns would start erroring after a minor update, and why does the leak justify that change on 14 through 17 rather than master only?

Counter-check: held. Message #1 states the new error but does not weigh it for stable branches, and says 14 and 15 are untested. Security fixes often change behavior in minor releases, so this is a question to answer and stays fixable.

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
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The patch changes when the RLS conflict check runs relative to the ON CONFLICT WHERE clause, to close a leak of hidden column values. |
| backpatchability | **yes** | The commit message asks for Backpatch-through: 14, and a hidden conflicting row with a false WHERE now raises an error. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | **yes** | The reorder is in ExecOnConflictUpdate() and ExecOnConflictSelect(), which run for every conflicting row of an upsert. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which existing upsert patterns start raising the RLS error after a minor update, and why does the leak justify that on stable branches instead of master only?
2. Does the leak reproduce on 14 and 15, and does the DO UPDATE part of the patch apply and pass the regression tests there?
3. Was the security team told before the public post, and is a coordinated release or CVE planned?
4. Do other paths, such as MERGE, evaluate user quals on an existing row before the RLS check?
5. What release-note text would warn users whose ON CONFLICT ... WHERE statements skip hidden rows today?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows row-level security enforcement and the ON CONFLICT code in nodeModifyTable.c, and who can judge behavior changes in stable branches. No committer has posted in the thread.
- **Liability:** The owner signs for a new error in existing upserts on every branch that gets the fix, for the life of those branches, and for whether the fix closes the leak fully.
- **Author follow-up:** No sign either way; the thread has one message, though the author names the missing stable-branch tests himself.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is 8.4 KB, touches one executor file plus rowsecurity tests, and fixes a leak for users who rely on RLS to hide rows. It has had no reply since 2026-10-06, and the backpatch decision needs a committer who knows the area.

## Appendix

### Refuted in the counter-check

- `security-1` Backpatched order change turns silent no-op upserts into errors. Refuted (not specific): Duplicate of standards-compat-1. It describes the same multi-tenant upsert that starts raising the RLS error after a minor update, and asks the same question about stable branches.
- `maintenance-fit-1` Backpatch to 14 changes upsert behavior in minor releases, untested there. Refuted (not specific): Duplicate of standards-compat-1, which already covers the behavior change in minor releases and the missing tests on 14 and 15.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P7] shares the RLS topic, but its recorded reason does not apply here. That reason was that a restriction which only hides row counts, already visible elsewhere, gives no real security gain. This proposal stops a leaky function in the ON CONFLICT WHERE clause from reading the column values of a row hidden by policy, which is a direct data leak. Message #1 points to no earlier discussion of this ordering, only to the commits that introduced it.
- standards-compat: 1 objection(s). The new order matches the CREATE POLICY docs and adds no syntax, GUC, catalog or dump change, so the only compatibility concern is the backpatch.
- replication-ondisk: 0 objection(s). The patch only changes the order of the RLS conflict check and the ON CONFLICT WHERE qual in the executor, so it adds no on-disk, WAL, catalog, wire protocol or extension API state. The behavior change in a backpatch is a real question, but it belongs to another angle; the code was not shown, so possible struct changes could not be checked.
- security: 1 objection(s). The leak is real. A leaky function and a plain predicate both see hidden column values, and putting the RLS check first closes both. The error that remains shows only that a conflicting key exists, which ON CONFLICT already shows. [P7] does not apply because that row-count issue was judged not to be a security bug. Two points are outside the design: the report was posted publicly instead of going to the security team first, and MERGE may need a separate check.
- maintenance-fit: 1 objection(s). From a maintenance and fit angle the core change is small and adds no hot-path cost for non-RLS users, since it only reorders two existing checks. It also adds no new API. Regression tests pin the order for both paths. Precedent [P7] does not apply: leaky functions in quals are a threat that RLS is meant to stop.
- already-exists: 0 objection(s). No existing feature, option or extension closes this leak: the WHERE clause runs inside the executor before the RLS check, and users cannot change that order. The proposal changes the order and adds no new mechanism, so there is no duplication to object to. [P7] concerned a different problem (row counts in EXPLAIN ANALYZE), so its reason does not apply.
- demand: 0 objection(s). This is a security fix for a row-level security leak with a working repro. It does not add a feature, so the demand question does not apply. Everyone who relies on RLS policies to hide rows gains, and the visible cost is one ordering change that matches the documented behavior. [P7] differs: Dean Rasheed judged the EXPLAIN row counts there not a security bug, while here hidden column values reach arbitrary functions.

### Precedents retrieved

- P1: [Allow self-conflicting INSERT INTO .. ON CONFLICT DO UPDATEs](https://commitfest.postgresql.org/patch/4142/) (cf, Returned with feedback, 2023-03-01) - judged not relevant; reason summarised from its thread
- P2: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P3: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant; reason summarised from its thread
- P4: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - judged not relevant; reason summarised from its thread
- P5: [minor error message enhance: print RLS policy name when only one permissive policy exists](https://commitfest.postgresql.org/patch/6180/) (cf, Withdrawn, 2026-09-23) - judged not relevant; reason summarised from its thread
- P6: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/1745/) (cf, Returned with feedback, 2019-06-28) - judged not relevant; reason summarised from its thread
- P7: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - used in the review; reason summarised from its thread
- P8: [INSERT .. ON CONFLICT DO SELECT](https://commitfest.postgresql.org/patch/1241/) (cf, Returned with feedback, 2018-01-22) - judged not relevant
- P9: [GUC to improve SECURITY INVOKER safety](https://commitfest.postgresql.org/patch/4121/) (cf, Withdrawn, 2023-02-02) - judged not relevant; reason summarised from its thread
- P10: [INSERT ... ON CONFLICT DO SELECT \[FOR ...\]](https://commitfest.postgresql.org/patch/5413/) (cf, Returned with feedback, 2025-08-05) - judged not relevant
- P11: [Multi-tenancy with RLS](https://commitfest.postgresql.org/patch/722/) (cf, Rejected, 2017-01-31) - judged not relevant
- P12: [Allow cluster owner to bypass authentication](https://commitfest.postgresql.org/patch/2244/) (cf, Returned with feedback, 2020-04-06) - judged not relevant
- P13: [pg_upgrade: fix memory leak in SLRU I/O code](https://commitfest.postgresql.org/patch/6462/) (cf, Withdrawn, 2026-02-05) - judged not relevant
- P14: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P15: [More isolated multi-tenancy with RLS](https://commitfest.postgresql.org/patch/357/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P16: [Code comments: why some text-handling functions are leakproof](https://commitfest.postgresql.org/patch/3510/) (cf, Returned with feedback, 2022-03-28) - judged not relevant
- P17: [Leaking memory in text_overlay function](https://commitfest.postgresql.org/patch/658/) (cf, Rejected, 2016-08-04) - judged not relevant
- P18: [conflict handling for COPY FROM](https://commitfest.postgresql.org/patch/2186/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P19: [Revert "Allow ON CONFLICT .. DO NOTHING on a partitioned table."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f05230752d53c4aa74cffa9b699983bbb6bcb118) (revert, Reverted, 2017-03-31) - judged not relevant
- P20: [Fix order of checking ICU options in initdb and create database](https://commitfest.postgresql.org/patch/3976/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P21: [\[PATCH\] Add UPDATE WHERE OFFSET IN clause](https://commitfest.postgresql.org/patch/3531/) (cf, Returned with feedback, 2022-04-08) - judged not relevant
- P22: [Track invalidation_reason in pg_replication_slots.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6ae701b4378db2284c77314560e95a93d0ba9484) (revert, Partially reverted, 2024-03-22) - judged not relevant
- P23: [Add checks on search_path for IMMUTABLE and SECURITY DEFINER functions](https://commitfest.postgresql.org/patch/3684/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [Allow pageinspect's bt_page_stats function to return a set of rows instead of a single row](https://commitfest.postgresql.org/patch/3713/) (cf, Returned with feedback, 2022-10-12) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 119 s; cost reported by the CLI: $0.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 59 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:51:05.137Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
