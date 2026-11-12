# Design pre-review: COPY FROM with RLS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6178](https://commitfest.postgresql.org/patch/6178/) (Needs review); authors: Jian He (stayreal)
- Thread: [COPY FROM with RLS](https://www.postgresql.org/message-id/flat/CACJufxFbmnoa5O-vL43DPTCGt6oagY4dXgKxy%3DrcD9-e9g0zEg%40mail.gmail.com), started 2025-10-28
- Messages read: 13 of 13
- Patch prose read: v16-0001-COPY-FROM-with-RLS.patch, v1-0001-COPY-FROM-with-RLS.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and levels 1a and 1b pass. The open points are at level 2 and are fixable. maintenance-fit-1 is decisive: COPY borrows executor state from a planned but never executed INSERT, the author calls it a hack in message #2, and no committer has commented. standards-compat-1, standards-compat-2 and standards-compat-3 are contract details that follow from that mechanism.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and only 3 of 13 messages were shown in full. Not checked: the v16 rule check, the test for conditional DO INSTEAD NOTHING, the single-row fallback, and cleanup of the started query on error.

## The proposal, restated

**Problem.** COPY FROM fails with "COPY FROM not supported with row-level security" on any table where RLS applies to the current user. Non-owner roles must use INSERT instead, which one tester measured at about 2.7 times slower than COPY on 1M rows.

**Proposed design.** CopyFrom() builds a dummy "INSERT INTO rel DEFAULT VALUES", runs it through parse analysis, the rewriter and the planner, and calls ExecutorStart() without ever running the plan. The executor state supplies the WITH CHECK OPTIONs from the table's policies, and COPY checks each row with ExecWithCheckOptions(). Because the rewriter is needed, COPY FROM errors on an RLS table that has rules, and v16 falls back to single-row inserts when a policy qual is volatile or has a subquery.

**User-visible surface.**
- COPY FROM now succeeds on tables with RLS enabled and applies INSERT policies to each row.
- New error "cannot copy to table" with DETAIL "COPY FROM is not supported on tables with rules when row-level security is enabled."
- Errors raised during policy checks show "CONTEXT: COPY t" without the line number.
- pg_stat_statements (track = 'all') and auto_explain show an extra nested "INSERT INTO schema.rel DEFAULT VALUES" entry.
- Docs: copy.sgml and the create_policy.sgml table list COPY ... FROM next to INSERT.

**What the author claims.**
- INSERT policies are enforced on each copied row, the same as for INSERT.
- The rewriter cannot be skipped, so rejecting rules on RLS tables is the only option.
- The line number is dropped on purpose, because ExecWithCheckOptions does not show row data for RLS violations.
- A volatile or subquery policy qual forcing single-row insert fixes the batching bug.

**Thread state.** Bryan Green, Yilin Zhang, solai v, Kirill Reshke and Joao Rito reviewed, and Green found a crash and a rule redirect that v15 fixed. Open on v16: the missing line number, the rule check rejecting ON DELETE rules, and the nested INSERT showing up in monitoring extensions.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-10-28; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. COPY depends on a planned but never executed dummy INSERT

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-1`

A later patch changes what ExecutorStart or ModifyTable initialization sets up, or how the rewriter applies policies. Plain INSERT keeps working and its tests pass. COPY FROM on RLS tables silently checks the wrong quals, or fails, and the break shows up only in a COPY-specific test, if one exists.

The author calls this "a hack" and is "not sure how valid it is" (message #2), and v1 calls the partitioned-table ModifyTable node one too (message #1). No other utility command plans a query only to borrow its executor state. Reviewers have already found a crash, a rule redirect, a batching bug and missing line numbers (messages #8, #10, #12), which shows how many paths this coupling touches. No committer in the thread has said this mechanism is acceptable.

Thread: [#1 (jian he, 2025-10-28)](https://www.postgresql.org/message-id/CACJufxFbmnoa5O-vL43DPTCGt6oagY4dXgKxy%3DrcD9-e9g0zEg%40mail.gmail.com), [#2 (jian he, 2026-05-10)](https://www.postgresql.org/message-id/CACJufxFdt0%2BR6C5ZByAK3DpMEXbb%3D3Hc0XyRTPvbjpc9sPt%3DgQ%40mail.gmail.com), [#8 (dbryan(dot)green(at)gmail(dot)com, 2026-08-27)](https://www.postgresql.org/message-id/2e3ed175-8618-46a9-847b-b7e54597aa83%40gmail.com), [#10 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#12 (Kirill Reshke, 2026-11-02)](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com)

**Question:** Why can COPY not build the WITH CHECK OPTIONs by calling the RLS code directly, as v1 did, without parsing, rewriting, planning and starting a dummy query?

Counter-check: held. The author calls the mechanism a hack in message #2 and no committer has commented on it. Messages #1 and #2 do answer the v1 question (v1 could not handle subquery policies), but the coupling to INSERT's executor setup remains.

### 2. Rule handling of COPY FROM depends on whether RLS applies to the role

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax, backpatchability · `standards-compat-1`

A table has an ON DELETE rule and RLS enabled. The owner loads it with COPY and it works, because RLS does not apply to the owner. A non-owner ETL role runs the same COPY and gets "cannot copy to table", and finds out only in production.

COPY FROM is documented to ignore rules, and v16 makes this depend on whether RLS applies to the current role. In message #12, an ON DELETE rule blocks COPY. The v16 commit message still says "any rule that would apply to an INSERT", but message #11 proposes checking rd_rules for any rule. The author says the rewriter cannot be skipped (message #9). That explains the restriction for INSERT rules, but no message gives a reason to reject rules for other events.

Thread: [#9 (jian he, 2026-09-01)](https://www.postgresql.org/message-id/CACJufxEPnQ_wx%3D-6P6h%3DhaxCO3CHeAijV7eeq3keUZ_Tve-1yw%40mail.gmail.com), [#10 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#11 (jian he, 2026-09-23)](https://www.postgresql.org/message-id/CACJufxEX3GD9XaG63M76fr-_QTt9ezVeSS3XvW%3DZGhgr7K9LDg%40mail.gmail.com), [#12 (Kirill Reshke, 2026-11-02)](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com)

**Question:** Can the check reject only rules that would change an INSERT, and can the docs state the exact contract for COPY FROM with RLS?

Counter-check: held. Message #12 shows an ON DELETE rule blocking COPY FROM on an RLS table, and no later message answers it. The author's rewriter argument in message #9 covers only rules that affect an INSERT.

### 3. Executor hooks and monitoring report an INSERT that never runs

fixable · level 2, rubric item 13 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

An audit or monitoring extension using ExecutorStart or ExecutorEnd hooks logs "INSERT INTO public.t DEFAULT VALUES" for every RLS COPY. A DBA or auditor reads it as an application statement and looks for an INSERT that did not happen.

Message #13 shows a second pg_stat_statements entry with rows = 0 and an auto_explain plan for the nested INSERT. Extensions that hook the executor now see a statement that is planned and started but never run, and nothing in the proposal tells them so. No message settles whether the dummy query should get a queryId or call the hooks at all.

Thread: [#2 (jian he, 2026-05-10)](https://www.postgresql.org/message-id/CACJufxFdt0%2BR6C5ZByAK3DpMEXbb%3D3Hc0XyRTPvbjpc9sPt%3DgQ%40mail.gmail.com), [#13 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110133016.61814-1-joaovictor.rito%40gmail.com)

**Question:** Can WITH CHECK OPTIONs be set up without passing a visible query through executor hooks, or how do extensions tell this internal query from a real one?

Counter-check: held. Message #13 shows the nested INSERT in pg_stat_statements and auto_explain, and leaves open whether it should get a queryId or call the hooks. No later message or v16 prose settles it.

### 4. RLS errors in COPY drop the line number that COPY errors show

fixable · level 2, rubric item 12 · design, psql UX conventions · `standards-compat-3`

A user loads a large file into an RLS table, and one row fails the policy or hits an error inside a policy function. The error shows only "CONTEXT: COPY t", so the user cannot find the row and has to bisect the file by hand.

Every other COPY FROM error reports the line number, and so do check constraint violations. The author's reason (message #6) is that ExecWithCheckOptions does not show row data for RLS violations. A line number is not row data, because the user supplied the file. Messages #10, #12 and #13 raise the point again, and v16 still drops the line number even for errors that are not policy violations.

Thread: [#6 (jian he, 2026-07-13)](https://www.postgresql.org/message-id/CACJufxHU3tetCM%2BQymxpj7-OLP8b6x3HJAnARadSrEAt64MHkg%40mail.gmail.com), [#10 (Bryan Green, 2026-09-10)](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#11 (jian he, 2026-09-23)](https://www.postgresql.org/message-id/CACJufxEX3GD9XaG63M76fr-_QTt9ezVeSS3XvW%3DZGhgr7K9LDg%40mail.gmail.com), [#12 (Kirill Reshke, 2026-11-02)](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com), [#13 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110133016.61814-1-joaovictor.rito%40gmail.com)

**Question:** What does reporting the input line number reveal that the user does not already have, given they supplied the file?

Counter-check: held. Messages #10, #12 and #13 each report the missing line number, and message #12 reproduces it on v16. The reason in message #6 is about row data, and a line number of the user's own file is not row data.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The dummy INSERT passes through planner and executor hooks, so extensions see a query that is started but never run. |
| SQL-standard conflict or non-standard syntax | **yes** | see objection standards-compat-1: Rule handling of COPY FROM depends on whether RLS applies to the role |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | Errors during policy checks show "CONTEXT: COPY t" without the line number, which \copy users also see. |
| security / privileges | **yes** | COPY FROM now enforces RLS INSERT policies, and reviewers found a rule redirect bypass and a batching bypass in earlier versions. |
| backpatchability | **yes** | see objection standards-compat-1: Rule handling of COPY FROM depends on whether RLS applies to the role |
| maintenance burden ("who maintains this in 10 years") | **yes** | COPY on RLS tables depends on what INSERT's rewriter, planner and ExecutorStart set up, so later changes there can break it. |
| performance on the hot path | **yes** | Each COPY FROM into an RLS table adds parse, rewrite and plan, and volatile or subquery policies force single-row inserts. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Is there a way to build the WITH CHECK OPTIONs, including subquery policies and per-partition ones, without planning and starting a dummy INSERT?
2. Does v16 reject any rule on the table or only rules that would change an INSERT, and which of the two will the docs promise?
3. Should the dummy INSERT get a queryId and call executor hooks, and if so, how do pg_stat_statements and audit extensions tell it from a user statement?
4. What does the input line number reveal that the user does not already have, given that the user supplied the file?
5. How much time do parse, rewrite and plan add to a small COPY FROM that runs often on an RLS table?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows the RLS rewriter code and the COPY FROM executor path; none of the visible messages or the CommitFest reviewer list shows one.
- **Liability:** The owner signs for RLS enforcement in COPY FROM matching INSERT in every case, and for a coupling to INSERT's executor setup that later executor or rewriter patches can break. A miss here is a security bug, with no time limit.
- **Author follow-up:** The author moved the entry through six CommitFests, rebased it, and fixed the crash and the rule redirect that reviewers reported, up to v16.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch removes a documented limit for roles that load data into RLS tables, and one tester measured the INSERT workaround at about 2.7 times slower. It touches only copy.c and copyfrom.c, but the mechanism is new and no committer has yet said it is acceptable.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Dummy INSERT exposes COPY to planner and executor hooks of extensions. Refuted (no failure scenario): A planner_hook that changes WITH CHECK OPTIONs would break a plain INSERT the same way, and EXPLAIN without ANALYZE already calls ExecutorStart and ExecutorEnd without ExecutorRun. The rest duplicates standards-compat-2.
- `security-1` Unexecuted dummy INSERT reaches executor hooks used by audit extensions. Refuted (not specific): Duplicate of standards-compat-2. COPY FROM already runs the INSERT permission check on the target table, so the only new effect named is the extra recorded statement.
- `maintenance-fit-2` Executor hooks see an INSERT that never runs. Refuted (not specific): Duplicate of standards-compat-2. Its added claim that hooks never see a query started but not run is inaccurate, because EXPLAIN without ANALYZE does that today.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent list has no earlier attempt at enforcing RLS in COPY FROM, and no thread message points to one. With no recorded earlier rejection or revert reason to apply, this angle has no objection.
- standards-compat: 3 objection(s). Three fixable contract issues; no standards or syntax conflict, since the proposal adds no new syntax or GUC.
- replication-ondisk: 1 objection(s). The patch adds no catalog, WAL, on-disk or wire-protocol change, so pg_upgrade and physical replication are unaffected. The material does not say whether logical replication tablesync, which also uses COPY FROM, changes behaviour under this patch.
- security: 1 objection(s). Thread messages #8 and #10 found an RLS bypass through a rule redirect, a quota policy bypass through batching and a crash any INSERT grantee could trigger. v15 and v16 address all three. Message #13 confirms that FORCE RLS, BYPASSRLS, column privileges and per-leaf partition policies match INSERT. Dropping the line number is a usability question, since the rows come from the user.
- maintenance-fit: 2 objection(s). Hot-path cost falls only on COPY into RLS tables, and message #13 shows that COPY stays much faster than the INSERT workaround.
- already-exists: 0 objection(s). The existing route is COPY into a temporary table followed by INSERT ... SELECT, which is the idiom the current docs point to. Message #13 measures it at about 2.7 times slower than the patched COPY, and no contrib module or extension enforces INSERT policies during COPY, so I have no already-exists objection to raise.
- demand: 0 objection(s). No demand objection. The patch removes a limitation that copy.sgml documents, and message #13 measures COPY at 440 ms against 1170 ms for the INSERT workaround on 1M rows; its cost falls only on COPY into RLS tables. The thread cites no -general or -bugs reports, so the demand evidence is in-thread testing only.

### Precedents retrieved

- P1: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant; reason summarised from its thread
- P2: [Document the limitations of row-level security](https://commitfest.postgresql.org/patch/2027/) (cf, Returned with feedback, 2019-04-04) - judged not relevant; reason summarised from its thread
- P3: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant; reason summarised from its thread
- P4: [Revert "Refactor CopyReadAttributes{CSV,Text}() to use a callback in COPY FROM"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=06bd311bce24083c76d9741ae89c98750aaf4b41) (revert, Reverted, 2024-02-14) - judged not relevant
- P5: [GUC to improve SECURITY INVOKER safety](https://commitfest.postgresql.org/patch/4121/) (cf, Withdrawn, 2023-02-02) - judged not relevant; reason summarised from its thread
- P6: [Parallel COPY FROM](https://commitfest.postgresql.org/patch/1266/) (cf, Returned with feedback, 2017-10-02) - judged not relevant; reason summarised from its thread
- P7: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [Revert my ill-considered change that made formrdesc not insert the correct relation rowtype OID into the relcache entries it builds.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca70c3cfda609d3b817c16b48f25c086c11b9e4c) (revert, Reverted, 2009-09-26) - judged not relevant
- P9: [Revert "Refactor CopyAttributeOut{CSV,Text}() to use a callback in COPY TO"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1aa8324b81fa4979ffdc6ccf81d560eac9446948) (revert, Reverted, 2024-02-06) - judged not relevant
- P10: [Revert fix missing call to table_finish_bulk_insert during COPY](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f5db56fc4d6e95c582b61c99328ea0702b869fa0) (revert, Reverted, 2019-07-01) - judged not relevant
- P11: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant; reason summarised from its thread
- P12: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant; reason summarised from its thread
- P13: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P14: [copy_srf (was COPY as a set returning function)](https://commitfest.postgresql.org/patch/869/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P15: [INSERT SELECT: BulkInsertState and table_multi_insert](https://commitfest.postgresql.org/patch/2553/) (cf, Returned with feedback, 2021-11-24) - judged not relevant
- P16: [Improvements in Copy From](https://commitfest.postgresql.org/patch/2642/) (cf, Withdrawn, 2020-09-27) - judged not relevant
- P17: [Revert a few small patches that were intended for version 19.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e1458f2f1b56cfe07b3e8fc99b223ea8c852d7ce) (revert, Reverted, 2025-06-11) - judged not relevant
- P18: [Revert "Propagate CTE property flags when copying a CTE list into a rule."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d1d2979852538d7021cc809a40ef127d59747697) (revert, Reverted, 2021-02-07) - judged not relevant
- P19: [Multi-tenancy with RLS](https://commitfest.postgresql.org/patch/722/) (cf, Rejected, 2017-01-31) - judged not relevant
- P20: [Patch to clean Query after rewrite-and-analyze](https://commitfest.postgresql.org/patch/2229/) (cf, Withdrawn, 2019-08-05) - judged not relevant
- P21: [Bug in query rewriter - hasModifyingCTE not getting set](https://commitfest.postgresql.org/patch/3134/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P22: [Vectored I/O in bulk_write.c](https://commitfest.postgresql.org/patch/4891/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P23: [A few new options for CHECKPOINT](https://commitfest.postgresql.org/patch/2842/) (cf, Withdrawn, 2021-01-20) - judged not relevant
- P24: [Fix warnings under ENFORCE_REGRESSION_TEST_NAME_RESTRICTIONS](https://commitfest.postgresql.org/patch/4451/) (cf, Rejected, 2023-07-19) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 125 s; cost reported by the CLI: $1.95
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 82 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:44:21.165Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
