# Design pre-review: Add `tsmatch` JSONPath operator for granular Full Text Search

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6436](https://commitfest.postgresql.org/patch/6436/); [thread](https://www.postgresql.org/message-id/flat/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), 8 of 8 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The goal is stated and plausible, but four design points need agreement on the list before code review. With no tsconfig, tsmatch reads default_text_search_config inside immutable jsonpath functions (standards-compat-1), and it resolves the config name with the caller's search_path (security-1). The unprefixed non-standard keywords and the dedicated predicate repeat concerns recorded in [P4] (prior-attempts-1, maintenance-fit-2). Demand is thin but does not stop the review alone (demand-1).

Not checked: Without the code, three things could not be checked: where the new item type is added, whether the new keywords stay usable as key names, and how tsmatch treats non-string values in lax and strict modes. Only 3 of 8 messages were shown.

## Prior attempts

- [Add jsonpath string method $.replace(from, to)](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com) (similar idea; Withdrawn, 2024-09-25). "The thing that bothers me about the original proposal is this: what if we add a new non-spec jsonpath method and then a new version of the spec adds a method with the same name, but not compatible with our method?" ([message](https://www.postgresql.org/message-id/906fe018-6999-4ae9-9a76-650088450e4b%40dunslane.net)) Different now: This adds a filter predicate with clauses for full text search, where [P4] added a string method. The naming concern and the request for a general function-call mechanism still apply, and the thread answers neither.
- [Revert "Reorganise jsonpath operators and methods"](https://www.postgresql.org/message-id/flat/CAM2+6=XjTyqrrqHAOj80r0wVQxJSxc0iyib9bPC55uFO9VKatg@mail.gmail.com) (related; Reverted, 2024-01-03). "The reordering of JsonPathItemType affects the binary on-disk compatibility of the jsonpath type, so we must not change it." Different now: [P1] reordered existing item types; this patch adds one. The addition is safe only if the new item type is appended without renumbering, and the patch prose does not say where it goes.

## Objections a reviewer is likely to raise

### 1. Default tsconfig makes immutable jsonpath functions depend on a GUC

A user builds an expression index or generated column on jsonb_path_exists(doc, '$.c[*] ? (@.body tsmatch "run")'), which is allowed because the function is immutable. Another session or a restore runs with a different default_text_search_config. Index scans and fresh evaluation then disagree, and the user sees missing rows long after the change.

The doc hunk says an omitted tsconfig falls back to default_text_search_config. jsonb_path_*, @@ and @? are immutable, and the project already split off jsonb_path_*_tz so that timezone-dependent datetime behaviour does not leak into immutable functions. One-argument to_tsvector(text) is marked stable for the same reason. The proposal does not say which of these existing rules tsmatch follows.

**Question:** Will tsmatch without tsconfig be rejected in the immutable functions, as datetime is handled with the _tz variants, or how else does it keep them immutable?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com); `standards-compat-1`</sub>

### 2. Unprefixed non-standard jsonpath keywords, an objection raised before

A later SQL/JSON revision defines a full text predicate, or a keyword named tsmatch, with different syntax or semantics. Postgres then has to choose between breaking jsonpath values that users stored in tables, views and index expressions, or departing from the standard for good. The conflict appears only when someone implements the new standard.

In [P4], a non-standard jsonpath method from the same author drew the objection that the spec could later add an incompatible item with the same name. That entry also included a request for a general way to call pg_proc functions from jsonpath. This patch adds three bare keywords (tsmatch, tsconfig, tsqparser) and a dedicated item type, with no prefix and no general mechanism. Messages #1 and #8 point to like_regex as precedent, but like_regex is defined by the standard, and the thread does not discuss naming or how the patch fits the [P4] direction.

Precedent: [Add jsonpath string method $.replace(from, to)](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com) (Withdrawn, 2024-09-25)

**Question:** Why should these non-standard keywords be unprefixed, and why is a dedicated predicate better than the general pg_proc call mechanism discussed for $.replace()?

<sub>fixable, level 2, prior rejection; messages [#1](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com); `prior-attempts-1`</sub>

### 3. Special-case tsearch hook instead of a general jsonpath function mechanism

Someone later adds a new tsquery parser function, or wants jsonpath to call another text function such as trigram similarity. Each case needs a new keyword, a new grammar rule, a new binary item and a new hard-coded mode string like "pl", "ph" or "w". Extensions cannot add their own.

The design maps short strings to four fixed tsquery functions inside jsonpath_exec.c, so jsonpath now carries a private copy of the tsearch parser list. In [P4], a reviewer asked for a general way to call suitable pg_proc functions from jsonpath instead of one-off methods, and the author started to sketch that design. This proposal returns to the one-off approach and does not say why the general mechanism was dropped. The accumulation test fails because ten such operators per release would be ten new grammar keywords.

Precedent: [Add jsonpath string method $.replace(from, to)](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com) (Withdrawn, 2024-09-25)

**Question:** Why is a dedicated tsmatch predicate better than a general way for jsonpath to call a pg_proc function on a JSON value?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#2](https://www.postgresql.org/message-id/CA%2Bv5N41Rr-18cjSsc3a6YvGTmnaXmfNiQM3e_gvdw76Yj7y6zA%40mail.gmail.com), [#8](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com); `maintenance-fit-2`</sub>

### 4. Config name resolved at run time from the caller's settings

An owner writes a row-level security policy with a tsmatch predicate. A user sets default_text_search_config, or sets search_path = myschema, pg_catalog with their own config named "english", so the stemming changes. The policy then returns a different result for that user, and nothing reports it.

The tsconfig string is kept as text in the jsonpath value and looked up when the path runs. The documentation says an omitted tsconfig falls back to the session's default_text_search_config. A regconfig argument to to_tsquery is resolved to an OID once, when the expression is created, so a stored expression always uses the same config. The proposal does not say how a jsonpath stored in a policy, view or index keeps one meaning for every caller.

**Question:** How does a tsmatch predicate stored in a policy or view get the same text search configuration whatever the caller's search_path and default_text_search_config?

<sub>fixable, level 2, security / privileges; messages [#8](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com); `security-1`</sub>

### 5. Demand rests on one author's workflow, and a workaround exists

The project adds a non-standard jsonpath predicate for a small group whose queries already work. Most users who need this today write a LATERAL query and apply `to_tsvector(...) @@ to_tsquery(...)`. If the memory claim does not hold, the main gain is shorter SQL, and the project carries the predicate for good.

The only stated demand is the author's own work (message #1). The thread has no -general or -bugs reports, no second user and no other database cited as having this. The one reviewer (message #5) reviewed the code without supporting the need. The memory cost of the workaround is claimed but not measured: the benchmark files are attached and no numbers appear in the thread.

**Question:** Which users besides the author have hit this, and do the attached benchmarks show the LATERAL workaround costing much more memory or time than tsmatch?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#5](https://www.postgresql.org/message-id/F1EAD15E-3002-49FC-8603-F2019F7FCD8A%40gmail.com); `demand-1`</sub>

## Questions for the author

1. Will tsmatch without tsconfig raise an error in the immutable jsonpath functions, as datetime does outside the _tz variants, or how else do they stay immutable?
2. Why are tsmatch, tsconfig and tsqparser unprefixed, given the concern in [P4] that a later SQL/JSON revision could define the same names differently?
3. Why is a dedicated tsmatch predicate better than the general way to call pg_proc functions from jsonpath that was requested in [P4]?
4. Do '$.tsmatch', '$.tsconfig' and '$.tsqparser' still parse as member accessors, and is the new item type appended without renumbering existing ones?
5. What time and memory do the attached benchmarks show for tsmatch against jsonb_path_query plus to_tsvector @@ to_tsquery on the same documents?

## Touch points

- **on-disk format / pg_upgrade:** The patch adds a new item type to the binary jsonpath format, which is stored in user tables.
- **ABI / extension API:** see objection maintenance-fit-2: Special-case tsearch hook instead of a general jsonpath function mechanism
- **SQL-standard conflict or non-standard syntax:** tsmatch, tsconfig and tsqparser are new jsonpath keywords that the SQL/JSON standard does not define.
- **planner behaviour change:** Immutable jsonb_path_* functions, usable in index expressions, would depend on default_text_search_config when tsconfig is omitted.
- **pg_dump / restore round-trip:** Stored jsonpath text that uses the new words as key names must still parse on restore.
- **security / privileges:** The tsconfig name is resolved at run time with the caller's search_path, which matters in policies and views.
- **maintenance burden ("who maintains this in 10 years"):** jsonpath gets a permanent non-standard predicate and a hard-coded list of tsquery parser modes.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** jsonb_path_query plus to_tsvector @@ to_tsquery already gives per-element matching in SQL.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the jsonpath grammar and binary format, with some knowledge of text search configuration lookup. The visible thread shows no committer discussing the design; the brief mentions only off-list comments.
- Priority: fix design first; audience niche; review cost medium. The only demand shown is the author's own work, and existing SQL can do the job. The 33 KiB patch touches the jsonpath grammar, scanner, executor and binary format, and the immutability and naming points must be settled before that review is useful.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
