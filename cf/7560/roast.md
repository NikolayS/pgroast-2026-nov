# Design pre-review: Proposal: Include CHECK constraint expression in violation error DETAIL

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7560](https://commitfest.postgresql.org/patch/7560/); [thread](https://www.postgresql.org/message-id/flat/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), 4 of 4 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The goal is plausible and the change is small, but the thread has not yet agreed that it is wanted. demand-1 and already-exists-1 survive: the only stated case is a saved lookup step, and the error already carries the constraint name as an error field. The verdict rests on absence of evidence in a three-week-old thread with one reviewer, so confidence is low. The level 2 objections that survived are fixable but were not evaluated.

Not checked: The code was not read. It could not be checked whether domain CHECK or partition constraint errors change, which deparse context and search_path rules the expression uses, or whether the expression length is bounded.

## Prior attempts

- [Mention column name in error messages](https://www.postgresql.org/message-id/flat/CANfkH5k-6nNt-4cSv1vPB80nq2BZCzhFVR5O4VznYbsX0wZmow@mail.gmail.com) (related; Returned with feedback, 2016-11-29). "If it isn't, we'd be best off to reject this patch rather than create an expectation among users that such context should be there." ([message](https://www.postgresql.org/message-id/4341.1478369632%40sss.pgh.pa.us)) Different now: P7 added column context to type coercion errors through a parse-analysis callback, so it missed run-time errors. This patch changes the executor path that raises table CHECK violations, so it has no comparable gap within that error class.

## Objections a reviewer is likely to raise

### 1. No demand shown beyond the author's own wish

A committer weighs a change to every CHECK violation message against one person's request. Nothing in the thread shows users struggling today, so the change risks churn for a problem nobody reported.

The only motivation is message #1: users "need to inspect the table with \d". No -general or -bugs reports, no other users and no comparison with other database systems appear. Message #2 reviews security, not need, and nobody in the thread discusses whether the change is wanted. The error already names the constraint and sets the constraint-name error field, so a client can call pg_get_constraintdef, and descriptive constraint names cover most cases.

**Question:** Which users or reports show that the constraint name plus \d or pg_get_constraintdef is not enough in practice?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), [#2](https://www.postgresql.org/message-id/CAGsfjr5TmF5LFt%3Dofrm5gDS-6-AamQqa12_eafHyL-r16R9UTw%40mail.gmail.com); `demand-1`</sub>

### 2. Constraint name and error fields already lead to the expression

Every CHECK violation, in every application and every server log, carries a longer DETAIL line. Users who already look up the constraint by name gain nothing from it. Logs fill with copies of expressions that are already stored in pg_constraint.

The error already names the constraint and carries the schema, table and constraint names as separate error fields. A client or user can read the condition from \d or from pg_get_constraintdef() on the matching pg_constraint row, and message #2 notes that any role can read these. The proposal therefore makes the lookup more convenient but does not make anything possible that was not possible before. The author's case is "saves users a step" (message #1), and the thread has no field reports or user requests showing that this step is a real burden.

**Question:** Which users cannot practically look up the expression from the constraint name or the error fields today, and how often does this happen?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/OS9P286MB648659674DE8406AE4742D8094642%40OS9P286MB6486.JPNP286.PROD.OUTLOOK.COM), [#2](https://www.postgresql.org/message-id/CAGsfjr5TmF5LFt%3Dofrm5gDS-6-AamQqa12_eafHyL-r16R9UTw%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Which users or reports show that the constraint name plus \d or pg_get_constraintdef() is not enough in practice?
2. Why must the server add the expression when clients already receive the schema, table and constraint names as error fields?
3. Which constraint violation errors change (table CHECK, domain CHECK, partition constraint), which do not, and why stop there?
4. How does the DETAIL behave for a CHECK expression of several kilobytes, and is truncation needed as for failing-row values?
5. Does the DETAIL follow the same deparse rules as pg_get_constraintdef(), including schema qualification under different search_path settings?

## Touch points

- **security / privileges:** The CHECK expression goes to clients and the server log, and is dropped only when the failing-row detail is suppressed.
- **maintenance burden ("who maintains this in 10 years"):** The DETAIL text of every CHECK violation now depends on ruleutils deparse output; v2 changes eleven expected files.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** The expression is already reachable from the constraint name through \d or pg_get_constraintdef().

## Who would commit it, and when

- Owner: silent. Someone who knows executor constraint checking and error reporting, and the ruleutils deparse code.
- Not now, show demand first: Reports from -general or -bugs, support cases, or several independent users on the thread who say the lookup step is a real burden, for example where the person reading the error or log cannot query the catalog. A committer saying the change is wanted would also settle it.
- Priority: needs champion; audience unclear; review cost low. The patch changes one executor file plus expected output, and adds no new setting or format. The thread has one reviewer, who discussed security only, so it needs a committer or more users to say whether the change is wanted.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
