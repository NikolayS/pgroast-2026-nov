# Design pre-review: Reject wholerow references where ColumnRef is forbidden

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7570](https://commitfest.postgresql.org/patch/7570/) (Needs review); authors: Paul Jungwirth (pjungwir)
- Thread: [Reject wholerow references where ColumnRef is forbidden](https://www.postgresql.org/message-id/flat/CA%2BrenyXoVubKLxGf%3Dopu4ty_OYbEC1TkpX5rGfwDQpUiAga-%3DA%40mail.gmail.com), started 2026-10-31
- Messages read: 3 of 3
- Patch prose read: v2-0002-Improve-error-message-expanding-foo.-in-invalid-l.patch, v2-0001-Reject-whole-row-references-where-column-referenc.patch, nocfbot-v2-0001-Reject-whole-row-references-where-column-referenc.patch, nocfbot-v2-0002-Improve-error-message-expanding-foo.-in-invalid-l.patch, v1-0001-Reject-whole-row-references-where-column-referenc.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived. The only one raised, replication-ondisk-1, was refuted by the critic. The proposal fixes server crashes that message #1 reproduces, Tom Lane agreed it is a bug in message #2, and v2 moves the check into expandNSItemVars as he asked. The remaining points are code-review questions about the v14 and v15 versions.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot confirm that the v14/15 patches cover every caller of expandNSItemVars, or that the check on master rejects no legal expansion. It also cannot confirm the critic's claim that stored defaults dump as plain column references.

## The proposal, restated

**Problem.** Column references are forbidden in column defaults and partition bounds, but a whole-row reference such as (ROW(t.*)).f1 gets past the check. Some statements crash the server, and others are accepted and crash on a later INSERT. Andres found it while reviewing FOR PORTION OF, and the author says it goes back at least to v14.

**Proposed design.** The patch moves the expression-kind check out of transformColumnRef into a shared helper, checkColumnRefAllowed. On master, expandNSItemVars calls the helper, so every expansion of a table's columns is checked; foo().* makes no Vars and stays allowed. On v14 and v15, the callers of expandNSItemVars call the helper instead, to keep the exported signature.

**User-visible surface.**
- ALTER TABLE ... ADD COLUMN, CREATE TABLE and ALTER COLUMN ... SET DEFAULT reject a default containing a whole-row reference such as (ROW(t.*)).f1.
- CREATE TABLE ... PARTITION OF rejects a partition bound containing a whole-row reference.
- Optional patch 0002: such references give "cannot use column reference" instead of "missing FROM-clause entry".
- checkColumnRefAllowed is a new helper declared in parser/parse_expr.h.

**What the author claims.**
- The listed statements crash the server or are wrongly accepted on master and back to at least v14.
- The fix allows foo().* and forbids (ROW(t.*)).f1.
- The v14/15 version keeps the ABI because the signature of expandNSItemVars does not change there.
- Patch 0002 fixes the more basic problem, because defaults and partition bounds have no FROM clause.

**Thread state.** Tom Lane agreed it is a bug but asked for the check to move from a caller of ExpandColumnRefStar down to something like expandNSItemVars. The author posted v2 doing that, plus optional patch 0002 and back-branch versions; no reply to v2 is visible.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds checkColumnRefAllowed to parser/parse_expr.h and, on master, changes how expandNSItemVars is called; v14 and v15 keep its exported signature. |
| SQL-standard conflict or non-standard syntax | **yes** | Defaults and partition bounds with a whole-row reference such as (ROW(t.*)).f1 now give an error instead of a crash. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The fix removes a server crash that any user allowed to create a table can cause. |
| backpatchability | **yes** | The commit messages say Backpatch-through: 14, with separate v14/15 patches that put the check in each caller of expandNSItemVars. |
| maintenance burden ("who maintains this in 10 years") | **yes** | On v14 and v15 the check sits in each caller of expandNSItemVars, the pattern Tom Lane called omission-prone in message #2. |
| performance on the hot path | **yes** | On master, expandNSItemVars runs an expression-kind check on every expansion of a table's columns; the cost is one test per expansion. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. On v14 and v15, which callers of expandNSItemVars do not call checkColumnRefAllowed, and why is each of them safe?
2. Which expression kinds besides column defaults and partition bounds forbid column references, and does a regression test reach each one through the new path?
3. With 0001 alone, what error message does CREATE TABLE p1 PARTITION OF p FOR VALUES IN ((ROW(p1.*)).f1) give?
4. Is patch 0002, which only changes an error message, meant to be back-patched to v14 as its commit message says?
5. Does the check inside expandNSItemVars on master reject any expansion of a table's columns that is legal today?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane, who knows the parser, reviewed v1 in message #2 and asked for the change that v2 makes. He has not replied to v2.
- **Liability:** The committer signs for a parser check back-patched to v14, including the per-caller version on v14 and v15, where a missed caller leaves a crash open.
- **Author follow-up:** The author posted v2 four days after Tom Lane's review, with separate patches for v14 and v15.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Few users write such defaults, but the statements in message #1 crash the server on every branch back to v14. The patches are small, touch only the parser and regression tests, and a committer who knows the area is engaged.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-1` Defaults already stored with whole-row references would break dump and upgrade. Refuted (wrong facts): The parser expands ROW(t.*) into one Var per column, so pg_dump would emit plain column references such as ROW(a, b). transformColumnRef already rejects those in a default on unpatched servers, so the patch should add no new restore or pg_upgrade failure.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P1] is about COPY FROM ... WHERE and generated columns, and its recorded reason does not apply to rejecting whole-row references in defaults and partition bounds. The only earlier discussion the thread names is Andres's report in message #1, which this patch fixes.
- standards-compat: 0 objection(s). This is a crash fix that changes no syntax, GUC, catalog or psql behaviour, and it brings whole-row references in line with the existing rule for column references. One edge case: a default already stored through the wrongly accepted path would fail on restore after upgrade, which release notes could cover.
- replication-ondisk: 1 objection(s). Apart from stored defaults, the fix is parse-time only. It has no WAL, on-disk or wire protocol change, and the v14/15 versions keep the expandNSItemVars signature.
- security: 0 objection(s). The patch adds a parse-time check that rejects whole-row references in defaults and partition bounds, and so removes a crash that any user allowed to create a table can cause; it adds no privilege, role or exposure. Defaults and partition bounds already stored on back branches are outside the security angle and belong to the upgrade and backpatch angle.
- maintenance-fit: 0 objection(s). This is a crash fix in the parser. Tom Lane, who knows the area, engaged and asked for the check to move into expandNSItemVars, and v2 does that on master. The new helper only centralizes a check that already exists, and its cost is an expression-kind test per table-column expansion, which is negligible. On v14/15 the check sits in each caller of expandNSItemVars, which is the omission-prone pattern Tom objected to in message #2. That risk is small because back branches rarely gain new callers, and whether every caller is covered is a code-review question, not a design one. [P1] is about COPY WHERE semantics and does not apply.
- already-exists: 0 objection(s). This is a crash fix for a parser check that misses whole-row references, and no existing feature, extension or hook can block such defaults or partition bounds. The patch extends the existing column-reference check through a shared helper, so it adds no second mechanism. P1 is about COPY WHERE semantics and does not apply.
- demand: 0 objection(s). This is a crash bug fix, so the demand case does not depend on user requests. Statements a user can type crash the server or store defaults that crash a later INSERT, and Tom Lane agreed it is a bug (message #2). Nobody uses the forbidden construct on purpose, so the patch costs users nothing. Precedent [P1] is about a feature request in COPY WHERE and does not apply.

### Precedents retrieved

- P1: [COPY WHERE clause generated/system column reference](https://commitfest.postgresql.org/patch/6179/) (cf, Withdrawn, 2026-01-08) - used in the review; reason summarised from its thread
- P2: [Allow foreign keys to reference a superset of unique columns](https://commitfest.postgresql.org/patch/3678/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P3: [Revert applying column aliases to the output of whole-row Vars.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec62cb0aac5ba31a82339606009ddbd7eb00e2ac) (revert, Reverted, 2022-03-17) - judged not relevant
- P4: [Improving default column names/aliases of subscript text expressions](https://commitfest.postgresql.org/patch/5446/) (cf, Rejected, 2025-02-05) - judged not relevant; reason summarised from its thread
- P5: [docs about FKs referencing partitioned tables](https://commitfest.postgresql.org/patch/2123/) (cf, Withdrawn, 2019-05-21) - judged not relevant; reason summarised from its thread
- P6: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P7: [Bug: Reading from single byte character column type may cause out of bounds memory reads](https://commitfest.postgresql.org/patch/3767/) (cf, Returned with feedback, 2022-09-08) - judged not relevant; reason summarised from its thread
- P8: [Extensible parsing](https://commitfest.postgresql.org/patch/3100/) (cf, Rejected, 2022-01-17) - judged not relevant; reason summarised from its thread
- P9: [Re: BUG #18741: Detaching a partition referencing a partitioned table fails with a trigger-related error](https://commitfest.postgresql.org/patch/5484/) (cf, Withdrawn, 2025-02-18) - judged not relevant; reason summarised from its thread
- P10: [postgres.exe doesn't output crash dump when it crashes before main()](https://commitfest.postgresql.org/patch/1525/) (cf, Rejected, 2019-11-12) - judged not relevant
- P11: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P12: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P13: [Allow parallel plan for referential integrity checks](https://commitfest.postgresql.org/patch/3539/) (cf, Returned with feedback, 2023-07-04) - judged not relevant
- P14: [LockHasWaiters() crashes on fast-path locks](https://commitfest.postgresql.org/patch/6732/) (cf, Withdrawn, 2026-09-03) - judged not relevant
- P15: [Extended statistics / estimate Var op Var clauses](https://commitfest.postgresql.org/patch/2831/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant
- P17: [Allow parallel plan for referential integrity checks](https://commitfest.postgresql.org/patch/4507/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [Fix standby may crash when switching-over in certain special cases](https://commitfest.postgresql.org/patch/5199/) (cf, Withdrawn, 2026-09-07) - judged not relevant
- P19: [Infinite Autovacuum loop caused by failing virtual generated column expression](https://commitfest.postgresql.org/patch/7100/) (cf, Withdrawn, 2026-08-19) - judged not relevant
- P20: [Add parallel counters to pg_stat_statements](https://commitfest.postgresql.org/patch/3787/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [Revert "Simplify addJsonbToParseState()"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=956cc4434c3a8e69813b325618402508d1dbdcd9) (revert, Reverted, 2015-05-27) - judged not relevant
- P22: [Add pg_upgrade check for invalid role references in pg_init_privs](https://commitfest.postgresql.org/patch/6856/) (cf, Withdrawn, 2026-06-17) - judged not relevant
- P23: [Fix hostaddr crash during non-blocking cancellation](https://commitfest.postgresql.org/patch/5741/) (cf, Withdrawn, 2025-07-07) - judged not relevant
- P24: [Desirability of psql \if expressions?](https://commitfest.postgresql.org/patch/1656/) (cf, Rejected, 2018-07-05) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 91 s; cost reported by the CLI: $1.15
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 51 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:41.044Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
