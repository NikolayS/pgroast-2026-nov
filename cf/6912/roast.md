# Design pre-review: SQL/JSON: JSON_TRANSFORM (SQL standard, subclause 6.44)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6912](https://commitfest.postgresql.org/patch/6912/); [thread](https://www.postgresql.org/message-id/flat/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection is fatal, and none is at level 1: the problem is stated and the demand objection was refuted. The four surviving objections are fixable level 2 points. standards-compat-2 and maintenance-fit-2 need someone who has the standard text and knows the SQL/JSON node design. maintenance-fit-1 and standards-compat-1 are design points the author can answer on the list.

Not checked: Only 3 of 8 messages and no code, documentation or tests were read. The grammar behavior for IGNORE after a function call, the json input and RETURNING support, and the handling of unsupported path steps could not be checked.

## Prior attempts

- [Add jsonb_translate(jsonb, from, to)](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com) (similar idea; Withdrawn, 2026-01-25). "Blindly replacing every instance of a single string regardless of context seems at best a niche use. Is there any reason not to make this an extension?" ([message](https://www.postgresql.org/message-id/ea9fa483-db55-48c8-a566-8a6d83690f78%40dunslane.net)) Different now: P1 replaced every value that equals a given string, with no context. This proposal targets values by jsonpath with SQL-standard syntax, which replies in P1 pointed to as the alternative.

## Objections a reviewer is likely to raise

### 1. Claimed portability rests on a subset of T883 nobody checked against the text

A user ports a query that uses an array position target such as '$.arr[0]', a filter, a RETURNING clause or a json input. It fails or returns jsonb with a changed key order. If Postgres later corrects a default to match the standard text, views and indexes created earlier behave differently after upgrade.

Portability is the main argument in message #1. The proposal does not cover array positions, filters, RETURNING or json input, although the existing JSON_QUERY and JSON_VALUE support RETURNING. The only reviewer used Oracle documentation without the standard (messages #2, #5), so defaults such as ERROR ON MISSING for an empty wildcard rest only on the author's reading (message #3). Deparse writes the defaults explicitly, which helps dumps, but a stored behavior that was wrong would still need a compatibility break to fix.

**Question:** Which parts of T883 does the patch cover, which parts are left out, and has someone with the standard text confirmed the chosen defaults?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN4CZFPN%3D887T0njqw%2BbOHUiO6ion6rR%3Db1%2BEcx%2BgAaCuL89Wg%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/CAFC%2Bb6rX08OYaQSGNWmGfjhBCE-Qf7tAtbXzv%3DD9K%2B%2B4W%3DLDBg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAN4CZFOfvdPy4J2oLL-fMph_A3YO7prHKv3bFQYDgD_DYj2VQA%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Reusing JsonExpr with a NULL path_spec breaks existing walkers

Code in core or an extension that handles JsonExpr assumes path_spec is set, as contain_mutable_functions_walker did. A JSON_TRANSFORM expression reaches it and the backend crashes. This happens at run time, in a path nobody tested, such as CREATE INDEX in message #5.

JSON_TRANSFORM is added as a JsonExpr op whose jsonpath lives in a new action field, while path_spec stays NULL. Review already found three crashes from this, in the expression walker, the mutator and clauses.c (messages #4 and #6), and each fix was a local special case. Every future consumer of JsonExpr has to remember the same special case, and nothing enforces it.

**Question:** Why is JSON_TRANSFORM a JsonExpr op and not its own node, and how are all JsonExpr consumers in core shown to handle a NULL path_spec?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/CAFC%2Bb6pz%2BQh889V14x8wm1usjNtS419JuaJQ1g5xeuYtUsz5OQ%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/CAN4CZFOfvdPy4J2oLL-fMph_A3YO7prHKv3bFQYDgD_DYj2VQA%40mail.gmail.com), [#6](https://www.postgresql.org/message-id/CAFC%2Bb6oAK75SV_1TRz9RDA7D-LeR9VDXSoLS1NSXdUSy4nYAYw%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. Second jsonpath walker duplicates jsonpath_exec semantics in the executor

A later release extends jsonpath, for example with a new accessor or changed lax unwrapping rules, and updates jsonpath_exec.c. The JSON_TRANSFORM walker in execExprInterp.c keeps the old behaviour without any error. Users then see JSON_TRANSFORM and jsonb_path_query() disagree on the same path, and find out from a bug report.

Patch 0002 adds its own path-to-steps converter and recursive rebuilder (jtSetPath, jtSetPathObject). Patch 0004 then adds lax/strict handling "similar to jsonb_path_query" as separate code. The walker accepts only key and wildcard steps, so array positions and filters, listed as open in the brief, would need a third copy of path logic. Nothing shown makes the two evaluators share code or tests that would fail when they diverge.

**Question:** Can the target-path matching reuse jsonpath_exec.c, or which shared tests fail when JSON_TRANSFORM and jsonb_path_query() disagree on a path?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#6](https://www.postgresql.org/message-id/CAFC%2Bb6oAK75SV_1TRz9RDA7D-LeR9VDXSoLS1NSXdUSy4nYAYw%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/CAFC%2Bb6oJR028bP-vVLo3OhWM_oFgg324z%2BNr3MZoJWTvLYwzXQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 4. IGNORE ON clause may not parse after a function-call value

A user writes `json_transform(j, replace '$.a' = to_jsonb(x) ignore on missing)`. The value ends in a function call, so the parser may treat IGNORE as the start of IGNORE NULLS and then fail on ON. The user gets a syntax error for a form the standard allows.

Patch 0003 resolves the IGNORE NULLS / IGNORE ON conflict by giving IGNORE_P a precedence that always shifts toward IGNORE NULLS after a function application. That keeps window syntax working, but it seems to send exactly the case where a value or PATH source ends in a function call down the wrong branch. The commit message does not say whether that case parses, and nothing in the thread tests it. Silencing a grammar conflict with precedence hides it from future gram.y changes.

**Question:** Does `replace '$.a' = some_func(x) ignore on missing` parse, and if so, which grammar rule makes it work despite the IGNORE_P precedence?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/CAFC%2Bb6pQxA0-YV8B-8KFub_bix_wdhA%2BoHrv7S1rYb2tQjjPzg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which parts of T883 does the patch cover, which does it leave out, and has someone with the standard text confirmed the chosen defaults?
2. Does JSON_TRANSFORM accept json input or a RETURNING clause, and are array position targets such as '$.arr[0]' and filter expressions rejected with an error?
3. Why is JSON_TRANSFORM a JsonExpr op with a NULL path_spec instead of its own node, and how were the other JsonExpr consumers in core checked?
4. Can target-path matching reuse jsonpath_exec.c, or which shared tests fail when JSON_TRANSFORM and jsonb_path_query() disagree on a path?
5. Does `replace '$.a' = some_func(x) ignore on missing` parse, and which grammar rule makes it work despite the IGNORE_P precedence?

## Touch points

- **on-disk format / pg_upgrade:** JSON_TRANSFORM is accepted in expression indexes, so a later change to its semantics affects index contents kept by pg_upgrade.
- **catalog changes:** Stored expressions in views, indexes and defaults carry the new JsonExpr op and its action fields.
- **ABI / extension API:** The patch changes JsonExpr in primnodes.h and adds parse nodes, which extensions that walk expression trees see.
- **SQL-standard conflict or non-standard syntax:** The patch adds JSON_TRANSFORM syntax, the keywords EXISTING and MISSING, and a precedence for IGNORE.
- **planner behaviour change:** Patches 0004 and 0005 change clauses.c, including the mutability check for the new op.
- **pg_dump / restore round-trip:** The new ruleutils deparse must reproduce every operation and behavior clause for stored expressions.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a separate jsonpath target walker in execExprInterp.c, JIT support, grammar and deparse code.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the SQL/JSON grammar and JsonExpr execution and who has the SQL standard text. Message #1 reports off-list guidance from Peter Eisentraut, but the visible messages show no committer posting.
- Priority: needs champion; audience a sizeable minority; review cost high. Five patches touch the grammar, parse analysis, the executor, JIT, clauses.c and ruleutils, and the semantics need a check against the standard that the only reviewer could not do. The goal has a standard feature and an on-list suggestion behind it, but no committer has posted in the visible thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
