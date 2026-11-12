# Design pre-review: POC: PLpgSQL FOREACH IN JSON ARRAY

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6557](https://commitfest.postgresql.org/patch/6557/); [thread](https://www.postgresql.org/message-id/flat/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), 28 of 28 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). Two committers asked for a generic mechanism and did not question the goal, so the demand objections do not stop review at level 1b. The design is not yet agreed. Tom Lane's concern in message #20 that the iterator should not know the target type has no reply after the options struct (maintenance-fit-1). The author also leaves the casting default open (maintenance-fit-2), and nobody discussed putting by-name binding under IN ARRAY (standards-compat-3).

Not checked: The code was not read, so the supported iterator modes, the handling of plain json input, and the real cost of moving coercion into PL/pgSQL could not be checked. Only 3 of 28 messages were shown in full; later messages are known through the objections.

## Prior attempts

- [iteration over record in plpgsql](https://www.postgresql.org/message-id/flat/CAFj8pRDCUgKQ57viHtcciybjyzqJn9=oA+0-reFHYF4LJSxjJg@mail.gmail.com) (similar idea; Returned with feedback, 2015-07-07). "While I don't have a problem with hstore_to_array, I don't think that row_to_array is a very good idea; it's basically encouraging people to throw away SQL datatypes altogether and imagine that everything is text." ([message](https://www.postgresql.org/message-id/25419.1427653646%40sss.pgh.pa.us)) Different now: P1 extended FOREACH with a key/value variable list over a row turned into a text array. This proposal assigns jsonb object fields by name to typed targets with coercion, so it does not reduce values to text.

## Objections a reviewer is likely to raise

### 1. Iterator API in SubscriptRoutines is still shaped for PL/pgSQL

An extension author adds iteration to a subscriptable type such as hstore. The constructor takes a target type and a five-value ForeachAIteratorMode, so the author must write coercion and composite conversion logic that only PL/pgSQL defines. Each type's iterator then casts slightly differently, and users find out when the same FOREACH behaves differently per type.

In message #20, Tom Lane said the point of using SubscriptRoutines is a general facility, and that iterators probably should not know the target type. The v20260803 patch still passes the result type and a coercion mode to the constructor (messages #26, #28), and PL/pgSQL remains the only caller. Message #25 says a separate coerce step is slower than FOR IN SELECT, but its own table shows the int case at about 48 ms against about 61 ms for FOR IN SELECT. A cleaner API would still be faster than today's FOR IN SELECT loop.

**Question:** Which ForeachAIteratorMode values must every iterator support, and why can coercion not stay in PL/pgSQL, given that the coerce-callback numbers still beat FOR IN SELECT?

<sub>fixable, level 2, ABI / extension API; messages [#20](https://www.postgresql.org/message-id/3648848.1782942004%40sss.pgh.pa.us), [#22](https://www.postgresql.org/message-id/CAFj8pRC8uULe7Zk%3Dj4DSUf0Xu2b%3D6_mEn%3D4PfKOYmP7rvES%3D1Q%40mail.gmail.com), [#24](https://www.postgresql.org/message-id/1474498.1784830027%40sss.pgh.pa.us), [#25](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAFj8pRAw4sBhS%2BRH2%2BABoaTqTBKRe_zqyy2dxmpPKdyF7uvxgg%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Casting defaults fixed under old syntax before strict or lax is decided

A release ships FOREACH x IN ARRAY jsonb_expr with the current mixed rule, where 3.14 casts to int but "3.14" fails. Users write functions that rely on this. A later release that adds lax mode or ERROR ON ERROR clauses cannot change the default without silently changing how those functions behave.

Messages #1 and #28 and the latest commit message leave the strict-versus-lax question open and invite discussion, but nobody in the thread has answered it. Message #21 shows that plain assignment from jsonb and FOREACH conversion already follow different rules. Reusing the existing IN ARRAY keyword means the default casting behaviour is part of a long-lived language contract, so it should be settled before commit and not after.

**Question:** Which casting default is proposed for jsonb elements, and how would later ERROR ON ERROR clauses be added without changing what existing functions do?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAFj8pRA%2BtuLvb4kZVEeLjNOK7-oqKwdgTuBu3ofNGkVU4RO2RQ%40mail.gmail.com), [#25](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com); `maintenance-fit-2`</sub>

### 3. IN ARRAY becomes the spelling for any subscriptable type

An extension type such as hstore adds the new iterator, and FOREACH k IN ARRAY h starts to loop over a value that is not an array, with its own target-binding and slicing rules. Readers of the function cannot tell from the statement whether the loop binds by position or by name.

The docs already say other FOREACH variants for non-array composites "may be added in future", which suggests a separate keyword. The latest patch drops IN JSON ARRAY and puts jsonb under IN ARRAY, though jsonb loops bind by name and reject SLICE > 0. Message #13 backs a generic iterator, but the thread has not settled the user-facing spelling.

**Question:** Why should non-array subscriptable types share the IN ARRAY spelling instead of a distinct FOREACH variant, given that their binding rules differ?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#13](https://www.postgresql.org/message-id/4166.1773289811%40sss.pgh.pa.us), [#16](https://www.postgresql.org/message-id/CAFj8pRDHdO9bLM4%3D8%3Dp0MF8rNpSKdw-F_%3DzG%3DYxWui8S4vn5RQ%40mail.gmail.com), [#28](https://www.postgresql.org/message-id/CAFj8pRCC1zCPeYfVJLFPKqtth5xHHf7vVTCLTd4H%3D5MSW2H5JA%40mail.gmail.com); `standards-compat-3`</sub>

### 4. JSON keys bind to local variable names, and misses become NULLs

A developer renames local variables x, y to v_x, v_y in a function that uses FOREACH x, y IN ARRAY. The loop still runs, but no key matches, so every iteration assigns NULL to both variables. Nobody sees a problem until wrong data shows up downstream.

The documentation hunk shows that missing keys give NULL with no error, as in the '{}' row. Local variable names are internal to PL/pgSQL today. This rule makes them part of the data contract, which no other PL/pgSQL target list does, since arrays and composites bind by position. Whether a missing key should raise an error belongs to the undecided strict and lax question.

**Question:** What stops a variable rename from turning every value into NULL, and why is no error raised when no key matches any target variable?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAFj8pRBUd1Ojs8O%2BcJY3qmruuM%3DQPby_XTJtL9d6YtUX15OvTQ%40mail.gmail.com); `standards-compat-2`</sub>

### 5. Benchmarks skip the existing idiom of building an array once, then FOREACH

A PL/pgSQL author who needs a fast loop over a jsonb array can already write `foreach x in array array(select jsonb_array_elements_text($1)::int)`, or array_agg over jsonb_populate_recordset for objects. That runs the executor once, not per row. If this idiom is close in speed, core carries a new iterator API in SubscriptRoutines and new FOREACH semantics for a small gain.

Every speed figure in the thread compares the patch only with FOR IN SELECT over jsonb_array_elements (message #1, #4, #25). The materialize-then-FOREACH idiom uses only existing features, and message #25 shows FOREACH over a ready array at about 11 ms against 25 ms for the patch. Name-based mapping of objects to a composite also exists already through jsonb_populate_recordset and jsonb_to_recordset. So the measured gain over the best existing approach is unknown, and so is the readability gain.

**Question:** How does the patch compare with FOREACH over an array built once by array(select jsonb_array_elements...) or array_agg over jsonb_populate_recordset, for scalars and composites?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAFj8pRD9Jjv%2Bm%3D6DP6vWWn5NnZeTPH9eoPFCnA1JE5hRKRDMxQ%40mail.gmail.com), [#4](https://www.postgresql.org/message-id/d6ac54b9-79f4-485b-a540-96c56cd0b991%40uni-muenster.de), [#25](https://www.postgresql.org/message-id/CAFj8pRDEXOarQ3Ym%3DndLtUm6P%2BL9htRv7kPdk6nnb4T12D-nFw%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Why can coercion not stay in PL/pgSQL, given that message #25 shows the separate coerce callback at about 48 ms against 61 ms for FOR IN SELECT in the int case?
2. Which ForeachAIteratorMode values does PL/pgSQL use, and which must every type's iterator support?
3. Which casting default will FOREACH over jsonb keep, and how would a later ERROR ON ERROR clause fit without changing existing functions?
4. Why should jsonb share the IN ARRAY spelling when it binds object fields by name and rejects SLICE greater than zero?
5. How does the patch compare with FOREACH over an array built once by array(select jsonb_array_elements...), for scalar and composite targets?

## Touch points

- **ABI / extension API:** SubscriptRoutines in subscripting.h gets an optional iterator constructor with a target type and a ForeachAIteratorMode option.
- **SQL-standard conflict or non-standard syntax:** FOREACH ... IN ARRAY accepts jsonb with by-name binding and a casting rule that differs from the SQL/JSON lax default.
- **maintenance burden ("who maintains this in 10 years"):** A new iterator contract for every subscriptable type, and casting semantics in PL/pgSQL that cannot change after release.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** FOR IN SELECT over jsonb_array_elements and FOREACH over an array built once already cover the use case.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane is a listed reviewer and commented on the iterator design in message #20, but has not replied to the last two versions. The owner must know PL/pgSQL execution and the subscripting support API.
- Priority: fix design first; audience unclear; review cost medium. The thread names no workload where jsonb loop overhead is the bottleneck, and the gain applies mainly to loops with a pure PL/pgSQL body. The patch is about 40 KiB in pl_exec.c and the subscripting code, and it adds an extension-facing API whose shape a committer questioned without a visible answer.

---

[roast-full.md](roast-full.md) has 1 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
