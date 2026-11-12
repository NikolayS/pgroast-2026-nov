# Design pre-review: Proposal: Include CHECK constraint expression in violation error DETAIL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7560](https://commitfest.postgresql.org/patch/7560/) (Needs review); authors: Taiki Koshino (taikikoshino)
- Thread: [Proposal: Include CHECK constraint expression in violation error DETAIL](https://www.postgresql.org/message-id/flat/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), started 2026-10-23
- Messages read: 4 of 4
- Patch prose read: v2-0001-Report-CHECK-expression-in-constraint-violation-d.patch, v1-0001-Report-CHECK-expression-in-constraint-violation-d.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The goal is plausible and the change is small, but the thread has not yet agreed that it is wanted. demand-1 and already-exists-1 survive: the only stated case is a saved lookup step, and the error already carries the constraint name as an error field. The verdict rests on absence of evidence in a three-week-old thread with one reviewer, so confidence is low. The level 2 objections that survived are fixable but were not evaluated.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. It could not be checked whether domain CHECK or partition constraint errors change, which deparse context and search_path rules the expression uses, or whether the expression length is bounded.

## The proposal, restated

**Problem.** When a CHECK constraint fails, the error names the constraint and shows the failing row but not the condition. Users must run \d or read the catalog to see what was violated.

**Proposed design.** ExecRelCheck returns the failed ConstrCheck instead of only its name. ExecConstraints then deparses ccbin and appends the expression to the existing DETAIL line. When the failing-row detail is suppressed for lack of privileges, the whole DETAIL, expression included, is omitted.

**User-visible surface.**
- Changed DETAIL text for CHECK violations: "Failing row contains (...). Check constraint expression: (age >= 18)."
- Same changed DETAIL text in the server log.

**What the author claims.**
- Showing the expression saves users a step of inspecting the table definition.
- The expression is not shown when the failing row is hidden, so no new information reaches clients or the log in that case.
- A reviewer argued any role can already read CHECK expressions from the catalog, so no permission check is needed; the author agreed.
- v2 updates regression expected output and all regression tests pass.

**Thread state.** One reviewer, Yash Jadhav, discussed security and found no extra control needed; the author agreed and posted v2 with updated test output. No committer has replied, and nobody has discussed whether the change is wanted.

## 2. Prior attempts

### Mention column name in error messages (related)

- **Link:** [https://commitfest.postgresql.org/patch/818/](https://commitfest.postgresql.org/patch/818/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CANfkH5k-6nNt-4cSv1vPB80nq2BZCzhFVR5O4VznYbsX0wZmow@mail.gmail.com](https://www.postgresql.org/message-id/flat/CANfkH5k-6nNt-4cSv1vPB80nq2BZCzhFVR5O4VznYbsX0wZmow@mail.gmail.com)
- **Date:** decided 2016-11-29; started 2015-06-29
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Tom Lane (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Tom Lane found that the callback only covers errors raised during parse analysis or constant folding, not run-time errors from non-constant expressions, and covering those would need executor changes with a likely performance cost. He argued that partial coverage would mislead users and that the cases it handles already have error cursor information. He proposed extending error cursor support into execution instead, and Michael Paquier returned the patch with feedback.
- **Reason (quote):** "If it isn't, we'd be best off to reject this patch rather than create an expectation among users that such context should be there." ([message](https://www.postgresql.org/message-id/4341.1478369632%40sss.pgh.pa.us))
- **What is different this time:** P7 added column context to type coercion errors through a parse-analysis callback, so it missed run-time errors. This patch changes the executor path that raises table CHECK violations, so it has no comparable gap within that error class.


## 3. Objections a reviewer is likely to raise

### 1. No demand shown beyond the author's own wish

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved) · `demand-1`

A committer weighs a change to every CHECK violation message against one person's request. Nothing in the thread shows users struggling today, so the change risks churn for a problem nobody reported.

The only motivation is message #1: users "need to inspect the table with \d". No -general or -bugs reports, no other users and no comparison with other database systems appear. Message #2 reviews security, not need, and nobody in the thread discusses whether the change is wanted. The error already names the constraint and sets the constraint-name error field, so a client can call pg_get_constraintdef, and descriptive constraint names cover most cases.

Thread: [#1 (Koshino Taiki, 2026-10-23)](https://www.postgresql.org/message-id/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), [#2 (Yash Jadhav, 2026-10-29)](https://www.postgresql.org/message-id/CAGsfjr5TmF5LFt%3Dofrm5gDS-6-AamQqa12_eafHyL-r16R9UTw%40mail.gmail.com)

**Question:** Which users or reports show that the constraint name plus \d or pg_get_constraintdef is not enough in practice?

Counter-check: held. The thread has one reviewer, who discussed security only, and no reports or other users asking for this. The thread is three weeks old, so this is a request to show demand and not a ground for rejection.

### 2. Constraint name and error fields already lead to the expression

fixable · level 1b, rubric item 5 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-1`

Every CHECK violation, in every application and every server log, carries a longer DETAIL line. Users who already look up the constraint by name gain nothing from it. Logs fill with copies of expressions that are already stored in pg_constraint.

The error already names the constraint and carries the schema, table and constraint names as separate error fields. A client or user can read the condition from \d or from pg_get_constraintdef() on the matching pg_constraint row, and message #2 notes that any role can read these. The proposal therefore makes the lookup more convenient but does not make anything possible that was not possible before. The author's case is "saves users a step" (message #1), and the thread has no field reports or user requests showing that this step is a real burden.

Thread: [#1 (Koshino Taiki, 2026-10-23)](https://www.postgresql.org/message-id/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), [#2 (Yash Jadhav, 2026-10-29)](https://www.postgresql.org/message-id/CAGsfjr5TmF5LFt%3Dofrm5gDS-6-AamQqa12_eafHyL-r16R9UTw%40mail.gmail.com)

**Question:** Which users cannot practically look up the expression from the constraint name or the error fields today, and how often does this happen?

Counter-check: held. CHECK violations already carry schema, table and constraint names as error fields, and any role can read the expression from the catalog. The author's only stated case is a saved step, which does not by itself answer the desirability question.

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
| security / privileges | **yes** | The CHECK expression goes to clients and the server log, and is dropped only when the failing-row detail is suppressed. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The DETAIL text of every CHECK violation now depends on ruleutils deparse output; v2 changes eleven expected files. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | The expression is already reachable from the constraint name through \d or pg_get_constraintdef(). |

## 5. Questions the author must answer before anyone reviews code

1. Which users or reports show that the constraint name plus \d or pg_get_constraintdef() is not enough in practice?
2. Why must the server add the expression when clients already receive the schema, table and constraint names as error fields?
3. Which constraint violation errors change (table CHECK, domain CHECK, partition constraint), which do not, and why stop there?
4. How does the DETAIL behave for a CHECK expression of several kilobytes, and is truncation needed as for failing-row values?
5. Does the DETAIL follow the same deparse rules as pg_get_constraintdef(), including schema qualification under different search_path settings?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows executor constraint checking and error reporting, and the ruleutils deparse code.
- **Liability:** The owner signs for the wording and length of every CHECK violation DETAIL, in clients and in the server log, and for its dependence on ruleutils output in later releases.
- **Author follow-up:** The author wrote in message #1 that he wants to continue developing the patch, and posted v2 with updated test output within a week of the review.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: Reports from -general or -bugs, support cases, or several independent users on the thread who say the lookup step is a real burden, for example where the person reading the error or log cannot query the catalog. A committer saying the change is wanted would also settle it.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** unclear
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch changes one executor file plus expected output, and adds no new setting or format. The thread has one reviewer, who discussed security only, so it needs a committer or more users to say whether the change is wanted.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) Only table CHECK errors gain the expression; similar constraint errors do not. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Every CHECK violation logs the full deparsed expression, with no opt-out. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Ties ruleutils deparse output to every CHECK violation message. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `maintenance-fit-1` Every CHECK violation now pays for deparsing, including caught errors. Refuted (no failure scenario): The existing code already builds the failing-row text before ereport, and a caught error already pays for a subtransaction abort. Nothing shows that one deparse adds a material cost to that; the log volume part is a duplicate of standards-compat-2.
- `maintenance-fit-3` Constraint name is already a structured field clients can resolve. Refuted (not specific): Duplicate of already-exists-1, which states the existing-alternative point without the unmeasured server cost claim.
- `demand-2` Every user pays for changed DETAIL text, few gain. Refuted (not specific): Any change to message wording alters expected output and text-matching log rules; Postgres does not keep DETAIL text stable, and clients have SQLSTATE and error fields for matching. The log length part is a duplicate of standards-compat-2.

### Review angles

- prior-attempts: 0 objection(s). No precedent records a rejection of showing the constraint expression. P7's partial-coverage reason fits this patch only weakly, because domain CHECK errors and partition constraint errors use other paths, and the thread has not raised that. P6 ended because the author did not reply, which does not apply here.
- standards-compat: 2 objection(s). The change to DETAIL text is not treated as a compatibility break, because message text is not a stable interface and the thread does not show anyone relying on it.
- replication-ondisk: 0 objection(s). The proposal only changes the text of the DETAIL field for CHECK violations; it adds no on-disk, WAL, catalog, protocol message or replication state. ExecRelCheck is internal to execMain.c, so the changed return type should not affect the extension ABI, though this cannot be confirmed without the code.
- security: 0 objection(s). The expression text is already readable by any role through pg_constraint and pg_get_constraintdef, and v2 drops it with the whole DETAIL when the failing row is hidden. With column-level privileges, a partial row shows only permitted values, and a referenced hidden column exposes only its name, which the catalog already shows. I found no new leak, escalation path or denial-of-service risk that I could tie to a concrete scenario.
- maintenance-fit: 3 objection(s). Objections are listed above.
- already-exists: 1 objection(s). One objection: the need is already met by existing mechanisms, so the proposal adds convenience only. The listed precedents concern column names in coercion errors, and their recorded reasons do not apply here.
- demand: 2 objection(s). Demand is not shown, but the idea is small and plausible. These are 'not now, show demand' objections, not a rejection.

### Precedents retrieved

- P1: [Revert structural changes to not-null constraints](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6f8bb7c1e9610dd7af20cdaf74c4ff6e6d678d44) (revert, Reverted, 2024-05-13) - judged not relevant
- P2: [clean up create|alter domain stmt incompatiable constraint error case and add regression test](https://commitfest.postgresql.org/patch/5385/) (cf, Withdrawn, 2024-12-09) - judged not relevant; reason summarised from its thread
- P3: [NLS: use gettext() to translate system error messages](https://commitfest.postgresql.org/patch/6172/) (cf, Rejected, 2026-03-24) - judged not relevant; reason summarised from its thread
- P4: [Improve amcheck to check UNIQUE constraint in btree index](https://commitfest.postgresql.org/patch/2976/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P5: [Bug: wrong relname in RemoveSubscriptionRel() error detail](https://commitfest.postgresql.org/patch/6621/) (cf, Withdrawn, 2026-07-12) - judged not relevant; reason summarised from its thread
- P6: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - used in the review; reason summarised from its thread
- P7: [Mention column name in error messages](https://commitfest.postgresql.org/patch/818/) (cf, Returned with feedback, 2016-11-29) - used in the review; reason summarised from its thread
- P8: [CHECK constraint deferrable Support](https://commitfest.postgresql.org/patch/4463/) (cf, Withdrawn, 2024-01-15) - judged not relevant; reason summarised from its thread
- P9: [Fix pg_get_constraintdef for NOT NULL constraints on domains](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09d09d4297b9acbc2848ec35e8bf030d6c1fae18) (revert, Partially reverted, 2024-11-27) - judged not relevant
- P10: [Revert "Add soft error handling to some expression nodes"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8ec5e0543b90372c8e6d5cc2cd3d2ff89ca0e82) (revert, Reverted, 2023-10-02) - judged not relevant
- P11: [Allow to warn or error on ... = NULL expressions](https://commitfest.postgresql.org/patch/1728/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P12: [Fix for REFRESH MATERIALIZED VIEW ownership error message](https://commitfest.postgresql.org/patch/1766/) (cf, Rejected, 2018-08-19) - judged not relevant
- P13: [pg_dump: fix NOT NULL constraint name comparison using makeObjectName](https://commitfest.postgresql.org/patch/6601/) (cf, Withdrawn, 2026-10-04) - judged not relevant
- P14: [Add errdetail() with PID and UID about source of termination signal](https://commitfest.postgresql.org/patch/6505/) (cf, Withdrawn, 2026-03-03) - judged not relevant
- P15: [Revert "Improve error message on snapshot import in snapmgr.c"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=be022908cfe2959dbb17c10475deeb492b8e6944) (revert, Reverted, 2023-09-14) - judged not relevant
- P16: [pg_get_constraintdef: Schema qualify foreign tables unless pretty printing is enabled](https://commitfest.postgresql.org/patch/3811/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P17: [revamp row-security tracking](https://commitfest.postgresql.org/patch/5397/) (cf, Withdrawn, 2025-10-23) - judged not relevant
- P18: [add DETAIL for wrong SCRAM password](https://commitfest.postgresql.org/patch/3022/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P19: [GUC to improve SECURITY INVOKER safety](https://commitfest.postgresql.org/patch/4121/) (cf, Withdrawn, 2023-02-02) - judged not relevant
- P20: [Go back to suppressing foreign_data DETAIL test output.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=09963cedced7ffb98a06298cc16305767fd2b4dd) (revert, Partially reverted, 2019-03-22) - judged not relevant
- P21: [Don't allow to EXECUTE empty queries (fixes Assert violation)](https://commitfest.postgresql.org/patch/5918/) (cf, Rejected, 2025-07-17) - judged not relevant
- P22: [Restrict EXPLAIN (ANALYZE) for RLS and security_barrier views](https://commitfest.postgresql.org/patch/4999/) (cf, Rejected, 2024-09-19) - judged not relevant
- P23: [Detect shared buffer access violations using mprotect()](https://commitfest.postgresql.org/patch/1765/) (cf, Returned with feedback, 2019-02-03) - judged not relevant
- P24: [Revert "Add GUC backtrace_on_internal_error"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=592a2283721f7143999364ef487f2b4993f5161d) (revert, Reverted, 2024-04-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 118 s; cost reported by the CLI: $1.19
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 89 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:09:50.770Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
