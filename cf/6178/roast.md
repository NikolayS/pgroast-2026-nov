# Design pre-review: COPY FROM with RLS

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6178](https://commitfest.postgresql.org/patch/6178/); [thread](https://www.postgresql.org/message-id/flat/CACJufxFbmnoa5O-vL43DPTCGt6oagY4dXgKxy%3DrcD9-e9g0zEg%40mail.gmail.com), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and levels 1a and 1b pass. The open points are at level 2 and are fixable. maintenance-fit-1 is decisive: COPY borrows executor state from a planned but never executed INSERT, the author calls it a hack in message #2, and no committer has commented. standards-compat-1, standards-compat-2 and standards-compat-3 are contract details that follow from that mechanism.

Not checked: The code was not read, and only 3 of 13 messages were shown in full. Not checked: the v16 rule check, the test for conditional DO INSTEAD NOTHING, the single-row fallback, and cleanup of the started query on error.

## Prior attempts

None found before 2025-10-28.

## Objections a reviewer is likely to raise

### 1. COPY depends on a planned but never executed dummy INSERT

A later patch changes what ExecutorStart or ModifyTable initialization sets up, or how the rewriter applies policies. Plain INSERT keeps working and its tests pass. COPY FROM on RLS tables silently checks the wrong quals, or fails, and the break shows up only in a COPY-specific test, if one exists.

The author calls this "a hack" and is "not sure how valid it is" (message #2), and v1 calls the partitioned-table ModifyTable node one too (message #1). No other utility command plans a query only to borrow its executor state. Reviewers have already found a crash, a rule redirect, a batching bug and missing line numbers (messages #8, #10, #12), which shows how many paths this coupling touches. No committer in the thread has said this mechanism is acceptable.

**Question:** Why can COPY not build the WITH CHECK OPTIONs by calling the RLS code directly, as v1 did, without parsing, rewriting, planning and starting a dummy query?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CACJufxFbmnoa5O-vL43DPTCGt6oagY4dXgKxy%3DrcD9-e9g0zEg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CACJufxFdt0%2BR6C5ZByAK3DpMEXbb%3D3Hc0XyRTPvbjpc9sPt%3DgQ%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/2e3ed175-8618-46a9-847b-b7e54597aa83%40gmail.com), [#10](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#12](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Rule handling of COPY FROM depends on whether RLS applies to the role

A table has an ON DELETE rule and RLS enabled. The owner loads it with COPY and it works, because RLS does not apply to the owner. A non-owner ETL role runs the same COPY and gets "cannot copy to table", and finds out only in production.

COPY FROM is documented to ignore rules, and v16 makes this depend on whether RLS applies to the current role. In message #12, an ON DELETE rule blocks COPY. The v16 commit message still says "any rule that would apply to an INSERT", but message #11 proposes checking rd_rules for any rule. The author says the rewriter cannot be skipped (message #9). That explains the restriction for INSERT rules, but no message gives a reason to reject rules for other events.

**Question:** Can the check reject only rules that would change an INSERT, and can the docs state the exact contract for COPY FROM with RLS?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#9](https://www.postgresql.org/message-id/CACJufxEPnQ_wx%3D-6P6h%3DhaxCO3CHeAijV7eeq3keUZ_Tve-1yw%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#11](https://www.postgresql.org/message-id/CACJufxEX3GD9XaG63M76fr-_QTt9ezVeSS3XvW%3DZGhgr7K9LDg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com); `standards-compat-1`</sub>

### 3. Executor hooks and monitoring report an INSERT that never runs

An audit or monitoring extension using ExecutorStart or ExecutorEnd hooks logs "INSERT INTO public.t DEFAULT VALUES" for every RLS COPY. A DBA or auditor reads it as an application statement and looks for an INSERT that did not happen.

Message #13 shows a second pg_stat_statements entry with rows = 0 and an auto_explain plan for the nested INSERT. Extensions that hook the executor now see a statement that is planned and started but never run, and nothing in the proposal tells them so. No message settles whether the dummy query should get a queryId or call the hooks at all.

**Question:** Can WITH CHECK OPTIONs be set up without passing a visible query through executor hooks, or how do extensions tell this internal query from a real one?

<sub>fixable, level 2, ABI / extension API; messages [#2](https://www.postgresql.org/message-id/CACJufxFdt0%2BR6C5ZByAK3DpMEXbb%3D3Hc0XyRTPvbjpc9sPt%3DgQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/20261110133016.61814-1-joaovictor.rito%40gmail.com); `standards-compat-2`</sub>

### 4. RLS errors in COPY drop the line number that COPY errors show

A user loads a large file into an RLS table, and one row fails the policy or hits an error inside a policy function. The error shows only "CONTEXT: COPY t", so the user cannot find the row and has to bisect the file by hand.

Every other COPY FROM error reports the line number, and so do check constraint violations. The author's reason (message #6) is that ExecWithCheckOptions does not show row data for RLS violations. A line number is not row data, because the user supplied the file. Messages #10, #12 and #13 raise the point again, and v16 still drops the line number even for errors that are not policy violations.

**Question:** What does reporting the input line number reveal that the user does not already have, given they supplied the file?

<sub>fixable, level 2, design; messages [#6](https://www.postgresql.org/message-id/CACJufxHU3tetCM%2BQymxpj7-OLP8b6x3HJAnARadSrEAt64MHkg%40mail.gmail.com), [#10](https://www.postgresql.org/message-id/ab913841-b458-4931-bb89-cee7fba9eb24%40gmail.com), [#11](https://www.postgresql.org/message-id/CACJufxEX3GD9XaG63M76fr-_QTt9ezVeSS3XvW%3DZGhgr7K9LDg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CALdSSPhkL%2BQofGqaN9S%2B4m3rwYHrS_POU%2Bo43gEvbzmuxqhqxA%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/20261110133016.61814-1-joaovictor.rito%40gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. Is there a way to build the WITH CHECK OPTIONs, including subquery policies and per-partition ones, without planning and starting a dummy INSERT?
2. Does v16 reject any rule on the table or only rules that would change an INSERT, and which of the two will the docs promise?
3. Should the dummy INSERT get a queryId and call executor hooks, and if so, how do pg_stat_statements and audit extensions tell it from a user statement?
4. What does the input line number reveal that the user does not already have, given that the user supplied the file?
5. How much time do parse, rewrite and plan add to a small COPY FROM that runs often on an RLS table?

## Touch points

- **ABI / extension API:** The dummy INSERT passes through planner and executor hooks, so extensions see a query that is started but never run.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Rule handling of COPY FROM depends on whether RLS applies to the role
- **psql UX conventions:** Errors during policy checks show "CONTEXT: COPY t" without the line number, which \copy users also see.
- **security / privileges:** COPY FROM now enforces RLS INSERT policies, and reviewers found a rule redirect bypass and a batching bypass in earlier versions.
- **backpatchability:** see objection standards-compat-1: Rule handling of COPY FROM depends on whether RLS applies to the role
- **maintenance burden ("who maintains this in 10 years"):** COPY on RLS tables depends on what INSERT's rewriter, planner and ExecutorStart set up, so later changes there can break it.
- **performance on the hot path:** Each COPY FROM into an RLS table adds parse, rewrite and plan, and volatile or subquery policies force single-row inserts.

## Who would commit it, and when

- Owner: silent. Needs a committer who knows the RLS rewriter code and the COPY FROM executor path; none of the visible messages or the CommitFest reviewer list shows one.
- Priority: needs champion; audience a sizeable minority; review cost medium. The patch removes a documented limit for roles that load data into RLS tables, and one tester measured the INSERT workaround at about 2.7 times slower. It touches only copy.c and copyfrom.c, but the mechanism is new and no committer has yet said it is acceptable.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
