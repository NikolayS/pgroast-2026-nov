# Design pre-review: skip check constraints verification for unaffected columns during UPDATE

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6270](https://commitfest.postgresql.org/patch/6270/) (Needs review); authors: Jian He (stayreal)
- Thread: [UPDATE run check constraints for affected columns only](https://www.postgresql.org/message-id/flat/CACJufxEtY1hdLcx%3DFhnqp-ERcV1PhbvELG5COy_CZjoEW76ZPQ%40mail.gmail.com), started 2025-12-01
- Messages read: 12 of 12
- Patch prose read: v8-0001-skip-unnecessary-check-constraint-verification-for-UPDATE.patch, v1-0001-UPDATE-run-check-constraints-for-affected-columns-only.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The decisive objection is demand-1: the proposal is a performance optimization, and the thread shows no benchmark, workload or user report. The patch adds a new rule on which CHECK enforcement depends, so a committer would likely ask for numbers before reading the design. Confidence is low because the verdict rests on absent evidence and only 3 of 12 messages are visible. The level 2 objections security-1 and maintenance-fit-1 survived but are not evaluated here.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm how v8 builds the updated-column set on each path, or whether it treats non-immutable CHECK expressions specially. Messages #4 to #12 were not visible, so some of the questions may already have answers there.

## The proposal, restated

**Problem.** Every UPDATE re-checks all CHECK constraints on the table, even ones that reference only columns the statement does not change. The author points to a Todo wiki item and an older thread. The thread does not quantify the cost or name the workloads that suffer from it.

**Proposed design.** During UPDATE, the executor skips a CHECK constraint when none of its referenced columns is updated, including columns changed through stored generated columns. It never skips when a BEFORE ROW UPDATE trigger exists, or when the constraint is NOT VALID, NOT ENFORCED, has a whole-row reference, or references no columns. v7 keeps separate cached check expressions for INSERT and UPDATE, following the generated-column pattern, so MERGE with both actions stays correct.

**User-visible surface.**
- New DEBUG1 log message: "skipping verification for constraint "%s" on table "%s"".
- Changed C API in executor.h and execnodes.h: ResultRelInfo check-expression caching, likely split into INSERT and UPDATE arrays as in v7.

**What the author claims.**
- A row that passed a constraint still passes it if no referenced column changes, so re-checking is unnecessary.
- Skipping is safe when generated columns can be skipped, since the same BEFORE ROW UPDATE trigger guard applies.
- Cross-partition UPDATE runs as DELETE plus INSERT, so it always checks all constraints.
- Reviewer Florin Irion calls the core optimization sound, including trigger, generated-column and whole-row handling.
- Reviewer solai v tested v8 and found no issues with UPDATE, MERGE, generated columns or triggers.

**Thread state.** Tom Lane and li carol raised the trigger case, which v2 handles. Florin Irion and solai v reviewed later versions positively; v8 is in Needs review and has moved through five CommitFests with no committer reply.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-12-01; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No measurement shows the skip saves meaningful time

fixable · level 1b, rubric item 3 · demand, maintenance burden ("who maintains this in 10 years"), performance on the hot path · `demand-1`

A committer pushes the patch, and from then on every user relies on new skip logic to keep CHECK constraints enforced on UPDATE. Most CHECK expressions are cheap comparisons, so typical users may see no measurable gain. A future bug in the skip rules would let invalid rows into tables, and nobody would notice until queries or applications hit bad data.

The only demand shown is a Todo wiki item and an older thread (message #1). No user report, workload or benchmark appears in twelve messages, and the reviews in messages #9 and #12 check correctness, not speed. The change adds a correctness-critical path for constraint enforcement, with MERGE, ON CONFLICT and replication cases, so a committer would likely ask what users gain for that risk.

Thread: [#1 (jian he, 2025-12-01)](https://www.postgresql.org/message-id/CACJufxEtY1hdLcx%3DFhnqp-ERcV1PhbvELG5COy_CZjoEW76ZPQ%40mail.gmail.com), [#9 (Florin Irion, 2026-03-06)](https://www.postgresql.org/message-id/CA%2BHEvJBziL%2BYmbs3zY4X6GNxdV3vdSw2djW9GSMRWVvV2nr2%3DQ%40mail.gmail.com), [#12 (solai v, 2026-09-08)](https://www.postgresql.org/message-id/CAF0whueiCbq%3Dpistq0a6QGZrrYF_q3sm4Ti7%2Bk41xs6rRCRg6w%40mail.gmail.com)

**Question:** Which constraint shapes and UPDATE workloads get a measurable speedup, and how large is it compared with the cost of the UPDATE itself?

Counter-check: held. The Todo item and the older thread show some lasting interest, but the thread has no benchmark, workload or user report. This fits a "show demand" outcome, which a measurement could answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch touches execReplication.c, and the thread does not say whether logical replication apply gets the skip. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch changes the ResultRelInfo check-expression cache in execnodes.h and declarations in executor.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | A skipped CHECK such as `check (owner = current_user)` changes which role can update a row. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | CHECK enforcement on UPDATE now depends on the command type and a complete updated-column set in every caller. |
| performance on the hot path | **yes** | The skip decision sits in the constraint check that every UPDATE on a table with CHECK constraints runs. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which constraint shapes and UPDATE workloads get a measurable speedup from the skip, and how large is it compared with the cost of the UPDATE itself?
2. Does v8 skip a CHECK whose expression is not immutable, such as `check (owner = current_user)`, and if so, how are users told?
3. Does logical replication apply get the skip in v8, and does ExecSimpleRelationUpdate still pass CMD_INSERT to ExecConstraints?
4. What fails loudly when a future caller passes CMD_UPDATE with an incomplete updated-column set?
5. Does v8 keep the separate INSERT and UPDATE cached arrays from v7, and how does it cover INSERT ... ON CONFLICT DO UPDATE?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows the executor's ModifyTable and constraint code would have to own it. Tom Lane asked one question in message #2, and the brief reports no committer reply to later versions.
- **Liability:** The owner signs for CHECK enforcement on every UPDATE, MERGE, ON CONFLICT and apply path, with no time limit. A bug in the skip rule stores invalid rows without an error.
- **Author follow-up:** The author posted eight versions, answered reviewers, and moved the entry through five CommitFests, which suggests the author stays for fixes.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A benchmark that shows a measurable UPDATE speedup for a named constraint shape, such as many or costly CHECK constraints on a wide table, compared with total UPDATE cost. A user report of CHECK re-evaluation as a bottleneck would also help.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** park
- **Audience:** unclear
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The idea follows the existing generated-column skip pattern and no surviving objection is fatal, but the thread shows no number for the gain. The patch is small (about 15 KB) but touches constraint enforcement in the executor, so review should wait for a benchmark.

## Appendix

### Held up in the counter-check, not listed above

- `security-1` (fixable, level 2) Skipped checks that depend on session state change who can update a row. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-1` (fixable, level 2) Skip rule depends on every caller passing the right command type. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-1` Visible behavior change for CHECK constraints that call non-immutable functions. Refuted (not specific): Duplicate of security-1. Both say the v8 exclusion list does not cover non-immutable CHECK expressions, and security-1 gives the more concrete case.
- `replication-ondisk-1` Logical apply path and the updated-column set behind the skip. Refuted (no failure scenario): If ExecSimpleRelationUpdate still passes CMD_INSERT, apply skips nothing. If it passes CMD_UPDATE, the apply worker marks every replicated column except unchanged TOAST values as updated, so the set covers each changed column. maintenance-fit-1 keeps the open question.
- `maintenance-fit-2` No measurement shows the saving is worth a new executor invariant. Refuted (not specific): Duplicate of demand-1. Its extra point about no committer owner belongs to committability and must not support a desirability objection.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list. Message #1 links an older thread and a Todo wiki item but does not say what that thread concluded or why the idea stalled, so no earlier rejection reason can be checked against this proposal.
- standards-compat: 1 objection(s). Apart from the behavior change above, the patch adds no syntax, GUC, catalog state or dumped object. Its only new user-visible output is a DEBUG1 message, so it raises no pg_dump, psql or SQL-standard issue.
- replication-ondisk: 1 objection(s). The proposal changes no on-disk format, WAL record, catalog or wire protocol, so pg_upgrade and physical standbys are unaffected. The ResultRelInfo change in execnodes.h is a normal major-version API change, and nobody has proposed backpatching it.
- security: 1 objection(s). Apart from the non-immutable expression case, the change adds no new privilege, role or exposed data. The DEBUG1 message reveals only constraint and table names to someone who can already update the table.
- maintenance-fit: 2 objection(s). Fit in core is reasonable, since the change follows the existing generated-column skip pattern; objections cover maintainability and unshown benefit.
- already-exists: 0 objection(s). Postgres has no feature, setting or documented idiom that skips CHECK evaluation for unchanged columns, and an extension cannot change ExecConstraints without forking executor code. The patch reuses the existing generated-column skip rule (ExecGetAllUpdatedCols, the BEFORE ROW UPDATE trigger guard, the I/U cached arrays) and does not add a second mechanism.
- demand: 1 objection(s). Only one objection. The Todo item is weak but real evidence of interest, and the benefit would reach all users with CHECK constraints, so a precise answer here is "not now, show measurements".

### Precedents retrieved

- P1: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P2: [Add support for ON UPDATE/DELETE actions on ALTER CONSTRAINT](https://commitfest.postgresql.org/patch/1533/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P3: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P4: [Fix permission tests for views/tables proven empty by constraint exclusion.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50c137487c96e629e0e5372bb3d1b5f1a2f71a88) (revert, Partially reverted, 2013-05-01) - judged not relevant
- P5: [Allow AddForeignUpdateTargets to add extra target expressions other than columns](https://commitfest.postgresql.org/patch/1819/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P6: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P7: [Change a constraint's index - ALTER TABLE ... ALTER CONSTRAINT ... USING INDEX ...](https://commitfest.postgresql.org/patch/2635/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [Optimizing NULL check on pkey column](https://commitfest.postgresql.org/patch/2699/) (cf, Returned with feedback, 2021-09-27) - judged not relevant; reason summarised from its thread
- P9: [Add ALWAYS DEFERRED option for CONSTRAINTs and CONSTRAINT TRIGGERs](https://commitfest.postgresql.org/patch/1319/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P10: [avoid bitmapOR-ing indexes for scan conditions implied by partition constraint](https://commitfest.postgresql.org/patch/2644/) (cf, Returned with feedback, 2021-02-01) - judged not relevant; reason summarised from its thread
- P11: [Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/4099/) (cf, Returned with feedback, 2024-10-08) - judged not relevant; reason summarised from its thread
- P12: [POC: Lock updated tuples in tuple_update() and tuple_delete()](https://commitfest.postgresql.org/patch/3752/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P13: [heap_insert() and heap_update() optimization](https://commitfest.postgresql.org/patch/1830/) (cf, Rejected, 2018-10-17) - judged not relevant
- P14: [automatic client-side column-level encryption](https://commitfest.postgresql.org/patch/3718/) (cf, Returned with feedback, 2024-06-27) - judged not relevant
- P15: [Extracting only the columns needed for a query](https://commitfest.postgresql.org/patch/2331/) (cf, Withdrawn, 2020-07-01) - judged not relevant
- P16: [\[PROPOSAL\] : Disallow use of empty column name in (column_name '') in ALTER or CREATE of foreign table.](https://commitfest.postgresql.org/patch/5184/) (cf, Rejected, 2025-03-26) - judged not relevant
- P17: [Fix index-only scan plans, take 2.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9a3ddeb519e8d91b670fa923619b212a9670389c) (revert, Partially reverted, 2022-01-03) - judged not relevant
- P18: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P19: [Modify the planner to better estimate caching effects](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Optimizer_.2F_Executor) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P20: [amcheck verification for GiST](https://commitfest.postgresql.org/patch/1800/) (cf, Returned with feedback, 2019-11-28) - judged not relevant
- P21: [Optimization for updating foreign tables in Postgres FDW](https://commitfest.postgresql.org/patch/162/) (cf, Returned with feedback, 2015-05-13) - judged not relevant
- P22: [Online checksums verification](https://commitfest.postgresql.org/patch/2369/) (cf, Returned with feedback, 2020-11-05) - judged not relevant
- P23: [Make VACUUM remove MultiXacts with updater XIDs more eagerly](https://commitfest.postgresql.org/patch/4095/) (cf, Withdrawn, 2023-01-26) - judged not relevant
- P24: [Revert "Avoid the creation of the free space map for small heap relations".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7db0cde6b58eef2ba0c70437324cbc7622230320) (revert, Reverted, 2019-05-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 143 s; cost reported by the CLI: $1.73
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 77 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:46:59.116Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
