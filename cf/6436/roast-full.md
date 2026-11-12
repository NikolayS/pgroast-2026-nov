# Design pre-review: Add `tsmatch` JSONPath operator for granular Full Text Search

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6436](https://commitfest.postgresql.org/patch/6436/) (Needs review); authors: Florents Tselai (florentst)
- Thread: [Patch: Add tsmatch JSONPath operator for granular Full Text Search](https://www.postgresql.org/message-id/flat/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), started 2026-01-26
- Messages read: 8 of 8
- Patch prose read: v6-0001-Add-tsmatch-JSONPath-operator-for-granular-Full-T.patch, v1-0001-Add-tsmatch-JSONPath-operator-for-granular-Full-T.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is stated and plausible, but four design points need agreement on the list before code review. With no tsconfig, tsmatch reads default_text_search_config inside immutable jsonpath functions (standards-compat-1), and it resolves the config name with the caller's search_path (security-1). The unprefixed non-standard keywords and the dedicated predicate repeat concerns recorded in [P4] (prior-attempts-1, maintenance-fit-2). Demand is thin but does not stop the review alone (demand-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, three things could not be checked: where the new item type is added, whether the new keywords stay usable as key names, and how tsmatch treats non-string values in lax and strict modes. Only 3 of 8 messages were shown.

## The proposal, restated

**Problem.** Users cannot run full text search on one element of a nested jsonb document. jsonb_to_tsvector flattens all text into one vector, so a match cannot be tied to sibling keys in the same object. The workaround of unnesting with jsonb_array_elements and LATERAL is verbose and, per the author, uses much memory.

**Proposed design.** The patch adds a boolean tsmatch predicate to the jsonpath grammar, modeled on like_regex, that tests a JSON string against a tsquery during path traversal. Optional tsconfig and tsqparser clauses, in any order, choose the text search configuration and the tsquery parser function. The compiled tsquery and config OID are cached in JsonPathExecContext for one execution; the predicate is not indexable beyond existing GIN path matching.

**User-visible surface.**
- jsonpath predicate `@ tsmatch "query"`, usable in filter expressions with && and other conditions.
- Optional jsonpath clause `tsconfig "regconfig"`; when omitted, default_text_search_config is used.
- Optional jsonpath clause `tsqparser "mode"` with values "pl", "ph", "w"; omitted means to_tsquery.
- New jsonpath keywords tsmatch, tsconfig, tsqparser, which change what jsonpath strings parse.
- New jsonpath item type in the binary jsonpath format.

**What the author claims.**
- tsmatch keeps the link between conditions in the same JSON object, which jsonb_to_tsvector loses.
- It avoids the verbose SQL and memory cost of unnesting with jsonb_array_elements and LATERAL joins.
- It follows the like_regex precedent, though the SQL/JSON standard defines no full text operator.
- Caching gives compile-once behavior per execution and keeps prepared statements stable.
- The author reports real-world need from his own work with JSON and full text search.

**Thread state.** Chao Li reviewed v3 in detail (naming, grammar, context reuse across array items, error message bug), and Andrew gave off-list comments; v6 claims to address both. No one has replied to v4 through v6, and no committer has discussed the design.

## 2. Prior attempts

### Add jsonpath string method $.replace(from, to) (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/5261/](https://commitfest.postgresql.org/patch/5261/)
- **Thread:** [https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com)
- **Date:** decided 2024-09-25; started 2024-09-15
- **Outcome:** Withdrawn (cf)
- **Who objected:** Andrew Dunstan, "David E(dot) Wheeler", Peter Eisentraut (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The thread does not state why the author withdrew the entry. David E. Wheeler noted that replace() is not in the SQL/JSON spec, and Andrew Dunstan objected that a future spec version could add an incompatible method with the same name, so he wanted non-spec method names prefixed or capitalized. Peter Eisentraut asked for a general way to call any suitable pg_proc function from jsonpath, and the author then began sketching that design instead.
- **Reason (quote):** "The thing that bothers me about the original proposal is this: what if we add a new non-spec jsonpath method and then a new version of the spec adds a method with the same name, but not compatible with our method?" ([message](https://www.postgresql.org/message-id/906fe018-6999-4ae9-9a76-650088450e4b%40dunslane.net))
- **What is different this time:** This adds a filter predicate with clauses for full text search, where [P4] added a string method. The naming concern and the request for a general function-call mechanism still apply, and the thread answers neither.

### Revert "Reorganise jsonpath operators and methods" (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAM2+6=XjTyqrrqHAOj80r0wVQxJSxc0iyib9bPC55uFO9VKatg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAM2+6=XjTyqrrqHAOj80r0wVQxJSxc0iyib9bPC55uFO9VKatg@mail.gmail.com)
- **Date:** decided 2024-01-03; started 2024-01-03
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 283a95da923605c1cc148155db2d865d0801b419. The reordering of JsonPathItemType affects the binary on-disk compatibility of the jsonpath type, so we must not change it. Revert for now and consider.
- **Reason (quote):** "The reordering of JsonPathItemType affects the binary on-disk compatibility of the jsonpath type, so we must not change it."
- **What is different this time:** [P1] reordered existing item types; this patch adds one. The addition is safe only if the new item type is appended without renumbering, and the patch prose does not say where it goes.


## 3. Objections a reviewer is likely to raise

### 1. Default tsconfig makes immutable jsonpath functions depend on a GUC

fixable · level 2, rubric item 12 · design, pg_dump / restore round-trip, planner behaviour change · `standards-compat-1`

A user builds an expression index or generated column on jsonb_path_exists(doc, '$.c[*] ? (@.body tsmatch "run")'), which is allowed because the function is immutable. Another session or a restore runs with a different default_text_search_config. Index scans and fresh evaluation then disagree, and the user sees missing rows long after the change.

The doc hunk says an omitted tsconfig falls back to default_text_search_config. jsonb_path_*, @@ and @? are immutable, and the project already split off jsonb_path_*_tz so that timezone-dependent datetime behaviour does not leak into immutable functions. One-argument to_tsvector(text) is marked stable for the same reason. The proposal does not say which of these existing rules tsmatch follows.

Thread: [#1 (Florents Tselai, 2026-01-26)](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#8 (Florents Tselai, 2026-04-05)](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com)

**Question:** Will tsmatch without tsconfig be rejected in the immutable functions, as datetime is handled with the _tz variants, or how else does it keep them immutable?

Counter-check: held. The doc hunk says an omitted tsconfig uses default_text_search_config, and jsonb_path_exists, @? and @@ are immutable and allowed in index expressions. The thread does not address this, and the _tz split is existing practice for the same problem.

### 2. Unprefixed non-standard jsonpath keywords, an objection raised before

fixable · level 2, rubric item 6 · prior rejection, SQL-standard conflict or non-standard syntax, on-disk format / pg_upgrade · `prior-attempts-1`

A later SQL/JSON revision defines a full text predicate, or a keyword named tsmatch, with different syntax or semantics. Postgres then has to choose between breaking jsonpath values that users stored in tables, views and index expressions, or departing from the standard for good. The conflict appears only when someone implements the new standard.

In [P4], a non-standard jsonpath method from the same author drew the objection that the spec could later add an incompatible item with the same name. That entry also included a request for a general way to call pg_proc functions from jsonpath. This patch adds three bare keywords (tsmatch, tsconfig, tsqparser) and a dedicated item type, with no prefix and no general mechanism. Messages #1 and #8 point to like_regex as precedent, but like_regex is defined by the standard, and the thread does not discuss naming or how the patch fits the [P4] direction.

**Precedents.**
- [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (Withdrawn, 2024-09-25); thread: [link](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com)

Thread: [#1 (Florents Tselai, 2026-01-26)](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#8 (Florents Tselai, 2026-04-05)](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com)

**Question:** Why should these non-standard keywords be unprefixed, and why is a dedicated predicate better than the general pg_proc call mechanism discussed for $.replace()?

Counter-check: held. like_regex is in the standard and tsmatch is not, so the like_regex model does not answer the naming concern. [P4] raised the same concern for a non-standard jsonpath addition, and the thread does not discuss prefixes or a general mechanism.

### 3. Special-case tsearch hook instead of a general jsonpath function mechanism

fixable · level 2, rubric item 11 · maintenance burden ("who maintains this in 10 years"), ABI / extension API, doesn't fit what Postgres is about (extension / external tool / already solved) · `maintenance-fit-2`

Someone later adds a new tsquery parser function, or wants jsonpath to call another text function such as trigram similarity. Each case needs a new keyword, a new grammar rule, a new binary item and a new hard-coded mode string like "pl", "ph" or "w". Extensions cannot add their own.

The design maps short strings to four fixed tsquery functions inside jsonpath_exec.c, so jsonpath now carries a private copy of the tsearch parser list. In [P4], a reviewer asked for a general way to call suitable pg_proc functions from jsonpath instead of one-off methods, and the author started to sketch that design. This proposal returns to the one-off approach and does not say why the general mechanism was dropped. The accumulation test fails because ten such operators per release would be ten new grammar keywords.

**Precedents.**
- [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (Withdrawn, 2024-09-25); thread: [link](https://www.postgresql.org/message-id/flat/9A942004-047D-4A4D-BAFC-B45FFDF9B595@gmail.com)

Thread: [#2 (Florents Tselai, 2026-02-01)](https://www.postgresql.org/message-id/CA%2Bv5N41Rr-18cjSsc3a6YvGTmnaXmfNiQM3e_gvdw76Yj7y6zA%40mail.gmail.com), [#8 (Florents Tselai, 2026-04-05)](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com)

**Question:** Why is a dedicated tsmatch predicate better than a general way for jsonpath to call a pg_proc function on a JSON value?

Counter-check: held. [P4] records a request for a general way to call pg_proc functions from jsonpath, in the same area and from the same author. The thread does not say why a dedicated predicate with hard-coded parser modes is preferred here.

### 4. Config name resolved at run time from the caller's settings

fixable · level 2, rubric item 7 · security / privileges · `security-1`

An owner writes a row-level security policy with a tsmatch predicate. A user sets default_text_search_config, or sets search_path = myschema, pg_catalog with their own config named "english", so the stemming changes. The policy then returns a different result for that user, and nothing reports it.

The tsconfig string is kept as text in the jsonpath value and looked up when the path runs. The documentation says an omitted tsconfig falls back to the session's default_text_search_config. A regconfig argument to to_tsquery is resolved to an OID once, when the expression is created, so a stored expression always uses the same config. The proposal does not say how a jsonpath stored in a policy, view or index keeps one meaning for every caller.

Thread: [#8 (Florents Tselai, 2026-04-05)](https://www.postgresql.org/message-id/CA%2Bv5N432xJwp7MYLU2Gau3XkUFE6cgKDbS7fOFycKvZUdH-9MQ%40mail.gmail.com)

**Question:** How does a tsmatch predicate stored in a policy or view get the same text search configuration whatever the caller's search_path and default_text_search_config?

Counter-check: held. The tsconfig name is stored as a string and resolved when the path runs, so the caller's search_path decides which configuration is used. That differs from a regconfig constant and goes beyond the immutability point, and the thread does not cover it.

### 5. Demand rests on one author's workflow, and a workaround exists

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds a non-standard jsonpath predicate for a small group whose queries already work. Most users who need this today write a LATERAL query and apply `to_tsvector(...) @@ to_tsquery(...)`. If the memory claim does not hold, the main gain is shorter SQL, and the project carries the predicate for good.

The only stated demand is the author's own work (message #1). The thread has no -general or -bugs reports, no second user and no other database cited as having this. The one reviewer (message #5) reviewed the code without supporting the need. The memory cost of the workaround is claimed but not measured: the benchmark files are attached and no numbers appear in the thread.

Thread: [#1 (Florents Tselai, 2026-01-26)](https://www.postgresql.org/message-id/CA%2Bv5N42aGz5sCa3HoURUK82b-PN6N6LtUj45vAcL7KEBd5i5fg%40mail.gmail.com), [#5 (Chao Li, 2026-03-02)](https://www.postgresql.org/message-id/F1EAD15E-3002-49FC-8603-F2019F7FCD8A%40gmail.com)

**Question:** Which users besides the author have hit this, and do the attached benchmarks show the LATERAL workaround costing much more memory or time than tsmatch?

Counter-check: held. The only demand in the thread is the author's own work in message #1, and the one review covers code only. No benchmark numbers appear in the thread, so the memory claim is unmeasured.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The patch adds a new item type to the binary jsonpath format, which is stored in user tables. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | see objection maintenance-fit-2: Special-case tsearch hook instead of a general jsonpath function mechanism |
| SQL-standard conflict or non-standard syntax | **yes** | tsmatch, tsconfig and tsqparser are new jsonpath keywords that the SQL/JSON standard does not define. |
| new GUC | no |  |
| planner behaviour change | **yes** | Immutable jsonb_path_* functions, usable in index expressions, would depend on default_text_search_config when tsconfig is omitted. |
| pg_dump / restore round-trip | **yes** | Stored jsonpath text that uses the new words as key names must still parse on restore. |
| psql UX conventions | no |  |
| security / privileges | **yes** | The tsconfig name is resolved at run time with the caller's search_path, which matters in policies and views. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | jsonpath gets a permanent non-standard predicate and a hard-coded list of tsquery parser modes. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | jsonb_path_query plus to_tsvector @@ to_tsquery already gives per-element matching in SQL. |

## 5. Questions the author must answer before anyone reviews code

1. Will tsmatch without tsconfig raise an error in the immutable jsonpath functions, as datetime does outside the _tz variants, or how else do they stay immutable?
2. Why are tsmatch, tsconfig and tsqparser unprefixed, given the concern in [P4] that a later SQL/JSON revision could define the same names differently?
3. Why is a dedicated tsmatch predicate better than the general way to call pg_proc functions from jsonpath that was requested in [P4]?
4. Do '$.tsmatch', '$.tsconfig' and '$.tsqparser' still parse as member accessors, and is the new item type appended without renumbering existing ones?
5. What time and memory do the attached benchmarks show for tsmatch against jsonb_path_query plus to_tsvector @@ to_tsquery on the same documents?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the jsonpath grammar and binary format, with some knowledge of text search configuration lookup. The visible thread shows no committer discussing the design; the brief mentions only off-list comments.
- **Liability:** The owner signs for three non-standard jsonpath keywords and a new on-disk item type that cannot be removed later. The owner also signs for the immutability of jsonpath functions that call text search code.
- **Author follow-up:** The author posted six versions, responded to review, and moved the entry to a new CommitFest in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The only demand shown is the author's own work, and existing SQL can do the job. The 33 KiB patch touches the jsonpath grammar, scanner, executor and binary format, and the immutability and naming points must be settled before that review is useful.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-3` (fixable, level 2) New keywords may stop existing jsonpath strings parsing after upgrade. Not listed: not among the strongest objections.
- `standards-compat-4` (fixable, level 2) Default parser and mode names depart from existing text search conventions. Not listed: not among the strongest objections.
- `already-exists-1` (fixable, level 1b) Existing SQL with jsonb_path_query and @@ already ties conditions together. Not listed: not among the strongest objections.

### Refuted in the counter-check

- `standards-compat-2` Unprefixed non-standard keywords may collide with future SQL/JSON spellings. Refuted (not specific): Duplicate of prior-attempts-1, which states the same naming concern with the same precedent and more thread detail.
- `replication-ondisk-1` Default text search config makes immutable jsonpath results session-dependent. Refuted (not specific): Duplicate of standards-compat-1. The standby and check constraint cases are further examples of the same immutability problem.
- `replication-ondisk-2` New item type changes the stored binary jsonpath format. Refuted (no failure scenario): The patch does not appear to reorder existing item types, and the v4 alignment fix seems to concern only the new item's own serialization. The remaining question is already asked in standards-compat-3.
- `maintenance-fit-1` Non-standard jsonpath keywords and item type become permanent. Refuted (not specific): Duplicate of prior-attempts-1. The added point from [P1], that a released item type cannot be removed, holds for any new jsonpath item.
- `maintenance-fit-3` Dependence on default_text_search_config inside immutable jsonpath functions. Refuted (not specific): Duplicate of standards-compat-1.
- `already-exists-2` A one-off FTS predicate where a general function-call design was asked for. Refuted (not specific): Duplicate of maintenance-fit-2, which makes the same general-mechanism argument from [P4].
- `demand-2` Permanent jsonpath format and naming cost for a narrow gain. Refuted (not specific): Duplicate of prior-attempts-1 for naming and permanence, and of demand-1 for thin demand. It adds no separate cost that falls on users who do not use tsmatch.

### Review angles

- prior-attempts: 1 objection(s). One objection only. The [P1] binary-format concern cannot be checked without the code, so it appears under prior attempts and is not raised as an objection.
- standards-compat: 4 objection(s). Objections listed; nothing further from the standards and compatibility angle.
- replication-ondisk: 2 objection(s). The proposal adds no WAL records, catalogs or wire protocol messages. Persistent state is limited to the binary jsonpath format and to indexes or constraints that use these functions.
- security: 1 objection(s). Apart from the config lookup, the feature adds no new privilege surface. Text search configs have no ACL, and the query string comes from the caller, as with to_tsquery.
- maintenance-fit: 3 objection(s). Committability: no committer has discussed the design on the list. Andrew's comments were off-list, so it is unknown who would own jsonpath grammar and tsearch coupling after commit.
- already-exists: 2 objection(s). Two objections raised; the existing SQL route can do the job, so the question is whether the convenience justifies the addition.
- demand: 2 objection(s). Two demand objections. Neither one shows that the user group is empty, so both are fixable with evidence.

### Precedents retrieved

- P1: [Revert "Reorganise jsonpath operators and methods"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0958f8f6bf60639a1d22d78d5b49a6092b11833d) (revert, Reverted, 2024-01-03) - used in the review
- P2: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P3: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - judged not relevant; reason summarised from its thread
- P4: [Add jsonpath string method $.replace(from, to)](https://commitfest.postgresql.org/patch/5261/) (cf, Withdrawn, 2024-09-25) - used in the review; reason summarised from its thread
- P5: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1366/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P6: [Remove 1MB size limit for tsvector](https://commitfest.postgresql.org/patch/1221/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P7: [Flexible configuration for full-text search](https://commitfest.postgresql.org/patch/1421/) (cf, Returned with feedback, 2019-02-04) - judged not relevant; reason summarised from its thread
- P8: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P9: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P10: [A function-based API for '@@' searches](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Improve text search error messages](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Text_Search) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P12: [Revert some careless search-and-replace: "ADD" in comment text should not be replaced with "ADD_P".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=12119188fe04cd20a6f1317f62886b685ae71c59) (revert, Reverted, 2005-12-29) - judged not relevant
- P13: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant; reason summarised from its thread
- P14: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P15: [Allow regex operations in PL/Perl using UTF8 characters in non-UTF8 encoded databases](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Server-Side_Languages) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P16: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P17: [Range phrase operator in tsquery](https://commitfest.postgresql.org/patch/1624/) (cf, Returned with feedback, 2018-11-30) - judged not relevant
- P18: [Add initdb option to initialize cluster with non-standard xid/mxid/mxoff.](https://commitfest.postgresql.org/patch/3634/) (cf, Rejected, 2023-03-21) - judged not relevant
- P19: [change regexp_substr first argument make tests more easier to understand](https://commitfest.postgresql.org/patch/4744/) (cf, Withdrawn, 2024-09-23) - judged not relevant
- P20: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P21: [Operation log for major operations](https://commitfest.postgresql.org/patch/4018/) (cf, Withdrawn, 2023-03-02) - judged not relevant
- P22: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant
- P23: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P24: [Named Operators](https://commitfest.postgresql.org/patch/4123/) (cf, Returned with feedback, 2023-07-04) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 19; wall time: 210 s; cost reported by the CLI: $2.78
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 129 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:51:20.620Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
