# Design pre-review: Detoast a column once per row instead of once per reference

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7423](https://commitfest.postgresql.org/patch/7423/); [thread](https://www.postgresql.org/message-id/flat/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is clear and demand is shown, so levels 1a and 1b pass. Message #3 reports wrong results on default settings in two cases, and the author has not yet replied. A custom scan returns 529 rows instead of 500 (replication-ondisk-1), and pg_column_size in a LATERAL call returns 320004 instead of 3679 (standards-compat-1). Both concern the safety rule the design rests on, and maintenance-fit-1 shows the hardcoded lists make such misses silent.

Not checked: Without the code, nobody could check that the executor's argument-position list is complete or that the JIT steps match the interpreter. The same holds for whether a parallel worker can follow the ParamExecData slot back-reference.

## Prior attempts

None found before 2026-09-30.

## Objections a reviewer is likely to raise

### 1. In-tree slots promise copy resets that extensions refilling slots in place break

An extension's custom scan uses the in-tree virtual slot ops and rewrites tts_values in place for each row, as was legal before. The four in-tree slot implementations declare resets_detoasted, so later rows read the first row's detoasted copy. Users get wrong query results in release builds and see no error.

Message #3 shows this case: 529 rows instead of 500 from a custom scan in a release build, and an assert failure in an assert build. The slot's promise is made by core, but extension code that never opted in decides whether it holds, and no current API forbids in-place refills. The reviewer's fix keeps tts_detoast_src in all builds and compares one pointer. The v1-0001 commit message says a matching address proves nothing because addresses are reused, so a buffer reused at the same address would still yield a stale copy.

**Question:** What check, valid in release builds, makes a stale copy impossible when out-of-tree code refills an in-tree slot in place and reuses the same datum address?

<sub>fixable, level 2, ABI / extension API; messages [#1](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com); `replication-ondisk-1`</sub>

### 2. Stored-form functions return different values depending on call position

A DBA runs a monitoring query with `lateral pg_column_size(t.col)` or wraps pg_column_compression in an extension function. After upgrade, with the default settings, it reports 320004 instead of 3679 and NULL compression. Nothing in the plan shows the change, so the DBA finds out only by comparing with a direct call.

Message #3 shows the LATERAL FROM case returning the decompressed size and a NULL compression method on v2 defaults. The config.sgml hunk documents that a function receiving the column as an argument now sees the fetched value. So whether a function sees the stored form depends on syntax, which no existing function contract has had. Open question 1 in message #1 says extensions cannot declare such functions, and the pg_proc flag or support request is not written.

**Question:** Will one check at every place that compiles function arguments, plus a declarable property for extension functions, be part of the series before it is committed?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com); `standards-compat-1`</sub>

### 3. Hardcoded function lists turn omissions into wrong answers and slower plans

A later release adds a function that reports the stored form, or one that reads only a slice, and no one adds it to ExecFuncReadsStoredForm() or ExecFuncReadsSliceOrSize(). The first returns a wrong value with detoast_reuse on. The second reads whole values where it read a few chunks. Users find out from wrong numbers or slower queries, and no test fails.

Message #1 admits that a function missing from the stored-form list silently returns a wrong answer, and that extensions cannot declare one. Message #3 shows the check is already bypassed for a LATERAL call, where pg_column_size returns 320004 instead of 3679. It also shows that slice readers such as "=" on text read 1843 buffers instead of 3. A second copy of the argument-position list in the planner is kept in step only by a comment, so each new expression node becomes a chore that reviewers are likely to miss.

**Question:** Can the stored-form and slice rules live in pg_proc or a support request, checked in one place, before this goes in?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com); `maintenance-fit-1`</sub>

### 4. Developer option defaults on and is read only at plan time

A user hits a wrong pg_column_size result or a slice-read regression and sets detoast_reuse = off as the documented escape. Prepared statements and cached PL/pgSQL plans keep the old plans and the old behaviour. The user concludes the setting does nothing.

Developer options are normally hidden debugging switches whose default is the plain behaviour. This one defaults on, changes what some functions return (message #3), and is the only user control. The config.sgml hunk says it applies only to plans made after the change. Message #1 does not say why a developer option is the right category for a default-on feature.

**Question:** Why is a default-on developer option the right category, and how does a user with cached plans turn the behaviour off?

<sub>fixable, level 2, new GUC; messages [#1](https://www.postgresql.org/message-id/CAHYEV2s-5Vn6aPWrtkOGu7MMrwaido-wVeAZg-gqE9LSYj%2Bhjw%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/71CC110B-6BDB-4CF5-B0D4-13378FCB8DFA%40gmail.com); `standards-compat-3`</sub>

## Questions for the author

1. What check, valid in release builds, prevents a stale copy when extension code refills an in-tree slot in place and the datum address is reused?
2. Will one stored-form check cover every place that compiles function arguments, including calls in FROM, and can extension functions declare this property before commit?
3. Will functions that read only a slice or the size get the stored datum unless a copy already exists, as message #3 asks?
4. Why is detoast_reuse a developer option when it defaults to on and is the only way for a user to turn the behaviour off?
5. How much more memory does a plan hold when several slots keep copies of wide values until the slot is refilled?

## Touch points

- **catalog changes:** Open question 1 proposes a pg_proc flag or support request for stored-form functions; the series does not include it yet.
- **ABI / extension API:** New fields on TupleTableSlot, TupleTableSlotOps, Plan, PlannedStmt and ParamExecData, plus a reset promise that in-tree slot types make for extensions.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-1: Stored-form functions return different values depending on call position
- **new GUC:** Adds detoast_reuse, a boolean developer option that defaults to on and is read at plan time.
- **planner behaviour change:** set_plan_detoast_reuse() in setrefs.c marks columns per plan node; the plan shape does not change.
- **security / privileges:** RLS and security-barrier quals are argument positions, so a stale copy after an in-place slot refill could reach them.
- **maintenance burden ("who maintains this in 10 years"):** Three hardcoded lists, a planner copy of the argument-position list, and new expression steps in the interpreter and JIT.
- **performance on the hot path:** A test runs at every argument position of every compiled expression; message #1 measures +0.22% to +0.5%.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows expression evaluation, the JIT expression code and the TupleTableSlot API. The visible thread has the author and one reviewer.
- Priority: fix design first; audience a sizeable minority; review cost high. Users with wide jsonb or text columns read several times per row gain 50% to 75% fewer buffer reads. The four-patch series, about 165 KiB, touches the executor, JIT and planner, and the wrong results in message #3 need a design answer first.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
