# Design pre-review: Proposal: JSON5 support in the JSON parsers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7021](https://commitfest.postgresql.org/patch/7021/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [Proposal: JSON5 support in the JSON parsers](https://www.postgresql.org/message-id/flat/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), started 2026-07-14
- Messages read: 2 of 2
- Patch prose read: v2-0001-Add-JSON5-comment-and-whitespace-support-to-the-J.patch, v2-0004-Add-JSON5-single-quoted-string-support-to-the-JSO.patch, v2-0002-Add-JSON5-trailing-comma-support-to-the-JSON-pars.patch, v2-0005-Add-JSON5-number-extensions-to-the-JSON-lexer.patch, v2-0003-Add-JSON5-unquoted-object-key-support-to-the-JSON.patch, v2-0008-Add-json5-type-documentation.patch, v2-0006-Add-JSON5-string-escapes-and-multi-line-strings-t.patch, v2-0007-Add-json5-data-type-with-cast-paths-to-json-jsonb.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The proposal fails at level 1b on demand. The only named consumer is the author's configuration format prototype, and the thread does not say if that proposal still needs this work (demand-1, already-exists-2). An extension can build the json5 type once the parser flag exists (already-exists-1). These are answerable, but the thread has only the author's two messages, so the verdict rests on absence of evidence.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: The effect of the implicit json to json5 cast on function and operator resolution could not be checked without the catalog entries. The level 2 objections on the type-versus-function choice, encoding-dependent validity and libpq size survived but were not evaluated here.

## The proposal, restated

**Problem.** Plain JSON is hard to write by hand, which the author met while prototyping a JSON-based configuration file format. The author wants JSON5 input available to SQL users and to extension authors through the common JSON parser.

**Proposed design.** A json5 flag on JsonLexContext makes the recursive-descent parser in src/common accept the JSON5 grammar, added one feature per commit; the incremental parser stays strict-JSON only. A new json5 type stores validated input verbatim and has casts to and from json and jsonb.

**User-visible surface.**
- New SQL data type json5, stored verbatim like json, with no functions or operators of its own.
- Casts: json5 to json and jsonb (assignment, normalizing, error on Infinity/NaN); json to json5 (implicit, binary coercible); jsonb to json5 via I/O.
- to_json, json_build_object, jsonb_agg and similar embed json5 values as JSON instead of as strings.
- C API: new json5 parameter on makeJsonLexContextCstringLen and a json5 flag on JsonLexContext; all existing callers pass false.
- libpq now links unicode_category, about 70 KB larger.

**What the author claims.**
- JSON5 support helps both SQL users and extension authors, apart from the configuration format proposal.
- v2 passes the full official json5-tests suite and was fuzzed against the json5 reference implementation.
- The strict JSON path shows only about 1% noise in release-build benchmarks.
- Changing a json column to json5 needs no table rewrite.
- The first commit, comments only (JSONC), is useful by itself, and the type commit can be dropped.

**Thread state.** Only the author has posted; v2 drops incremental parser support after a comment from Andrew in another thread. Tristan Partin is listed as reviewer but has not posted in this thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-14; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. No user demand shown for a json5 SQL type

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds json5 as a core type with permanent catalog entries and casts, but few users store JSON5 in tables. The type cannot be removed later without breaking dumps and upgrades. Maintainers then carry an ECMAScript 5.1 grammar in the backend for a small group.

The only use case in message #1 is the author's configuration format prototype, and message #2 adds no users. Nobody else has posted, and the listed reviewer has not either. The thread names no -general or -bugs requests and no other databases with a JSON5 column type. A committer would likely ask, as in [P1], why a niche syntax cannot first live in an extension and prove its demand there.

**Precedents.**
- [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (Withdrawn, 2026-01-25); thread: [link](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com)

Thread: [#1 (Zsolt Parragi, 2026-07-14)](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2 (Zsolt Parragi, 2026-10-01)](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com)

**Question:** Which users or applications store hand-written JSON5 in Postgres today, and what do they do now, for example strip comments in the client before insert?

Counter-check: held. Only the author has posted, and the one use case is the author's own prototype. P1 is about a niche function, so it is weak support, but the objection stands without it.

### 2. Clients can turn JSON5 into JSON before it reaches the server

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-2`

An application that keeps hand-written JSON5 files converts them to JSON with a JSON5 library on the client and inserts the result into json or jsonb. That path works today on every supported release. The in-core grammar adds lexer paths in src/common and about 70 KB to libpq, and no frontend caller is shown to use them.

The only consumer named in message #1 is the configuration format prototype, and the thread does not say whether that proposal still depends on this work. JSON5 parsers already exist outside Postgres, and message #2 tests against the reference one. Extension authors who want JSON5 can bundle such a parser. Core placement is mainly a convenience unless a core caller, such as a server configuration file, needs it.

Thread: [#1 (Zsolt Parragi, 2026-07-14)](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2 (Zsolt Parragi, 2026-10-01)](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com)

**Question:** Which in-core caller, backend or frontend, needs to parse JSON5 and cannot convert the input to JSON first?

Counter-check: held. The thread names one consumer, the configuration format prototype, and does not say if it still depends on this work. No in-core caller sets the json5 flag except the new type.

### 3. The json5 type can be built as an extension on today's hooks

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

Core ships json5 as a built-in type with no functions or operators. Users who store configuration in it must cast to json or jsonb in every query. The project then keeps the type OID, the casts and the docs through every release and pg_upgrade, even if the configuration format proposal goes another way.

Patch 0007 adds an input function, verbatim storage, a binary-coercible cast from json, and normalizing casts to json and jsonb. An extension can already do all of this with create type, create cast ... without function and create cast with a function. For types outside the built-in OID range, to_json and json_build_object already use a cast to json if one exists. Message #1 says the type commit can be dropped, so the core need left is the parser flag.

**Precedents.**
- [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (Withdrawn, 2026-01-25); thread: [link](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com)

Thread: [#1 (Zsolt Parragi, 2026-07-14)](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2 (Zsolt Parragi, 2026-10-01)](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com)

**Question:** What does a built-in json5 type give users that an extension type, using the patched parser and the existing cast-to-json lookup, cannot?

Counter-check: held. An extension can create the type, the binary-coercible cast and the normalizing casts, and the JSON builders already use a cast to json for non-built-in types. P1 concerns a different function and must not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The json5 type stores validated text verbatim, and that stored form must stay valid across pg_upgrade. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | v2-0007 adds rows to pg_type, pg_proc and pg_cast, including an implicit json to json5 cast. |
| ABI / extension API | **yes** | makeJsonLexContextCstringLen gets a new json5 parameter and JsonLexContext a new flag. |
| SQL-standard conflict or non-standard syntax | **yes** | json5 is a non-standard type name and input syntax next to the SQL/JSON types. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Raw non-ASCII keys and whitespace are valid only in UTF8 databases, so a dump can fail to restore elsewhere. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The shared parser in src/common/jsonapi.c carries a second grammar, and libpq grows by about 70 KB. |
| performance on the hot path | **yes** | The patches change the JSON lexer used by all json and jsonb input; message #2 reports about 1% noise. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | An extension could provide the json5 type, and clients can convert JSON5 to JSON today. |

## 5. Questions the author must answer before anyone reviews code

1. Does the configuration format proposal still depend on this work, and if so, on which of the eight commits?
2. Which in-core caller, backend or frontend, other than the json5 type, would set the json5 flag?
3. Which users store hand-written JSON5 in Postgres today, and what do they do now instead?
4. What does a built-in json5 type give users that an extension type using the patched parser and the existing cast-to-json lookup cannot?
5. If no frontend caller enables json5, can the json5 code and unicode_category stay out of the libpq build?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who maintains the common JSON parser (src/common/jsonapi.c) and the json/jsonb types would have to own it; none has posted in this thread.
- **Liability:** A permanent built-in type with its OID and casts, an ECMAScript 5.1 grammar in the shared parser, and a Unicode table dependency in libpq, with no end date.
- **Author follow-up:** The author posted v2 about 11 weeks after v1 with fuzzing fixes and benchmarks, and moved the entry to the next CommitFest.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A committed or agreed in-core consumer of the json5 flag, such as the configuration format, or requests from independent users who store JSON5 in Postgres. A statement from a JSON parser maintainer that the flag is wanted for extensions would also help.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** unclear
- **Maintenance burden:** high
- **Review cost:** high
- **Why:** Eight patches of about 150 KiB change the shared lexer and add a permanent type, while the thread names one consumer and has no reviewer reply. Splitting the parser flag from the type and tying the flag to a named consumer would give reviewers something smaller to decide.

## Appendix

### Held up in the counter-check, not listed above

- `standards-compat-1` (fixable, level 2) A new data type where a parsing function or input option would do. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) Input validity depends on database encoding, which breaks dump and restore. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) Every libpq user pays about 70 KB for an unused feature. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `standards-compat-3` Stored json5 values can fail later in casts and JSON builders. Refuted (no failure scenario): Infinity and NaN are part of JSON5, and the docs hunk documents the cast error. The json type already accepts \u0000, which the cast to jsonb rejects, so this copies existing practice.
- `replication-ondisk-1` Validity of stored json5 values depends on database encoding and Unicode tables. Refuted (not specific): Duplicate of standards-compat-2. The added Unicode-update case is speculative, because new Unicode versions mostly assign new characters, which makes more input valid.
- `replication-ondisk-2` New parameter on makeJsonLexContextCstringLen breaks extension builds. Refuted (no failure scenario): Postgres gives no source compatibility for backend C APIs across major versions, and this constructor's signature has changed before. A compile error is loud and the fix is one argument.
- `maintenance-fit-1` Permanent core json5 type without shown demand or functions. Refuted (not specific): Duplicate of demand-1 for the missing demand and of already-exists-1 for the extension route. Those two state each point more concretely.
- `maintenance-fit-3` Two grammars in one shared parser, and the parsers diverge. Refuted (no failure scenario): The scenario of a later fix missing the json5 branches is not concrete, and the patch set carries the full json5-tests suite on those branches. The incremental parser stays strict because of the area maintainer's comment, and committer ownership belongs under committability.
- `demand-2` Everyone pays libpq size and parser complexity for an optional syntax. Refuted (not specific): Duplicate of maintenance-fit-2 for the libpq size cost and of already-exists-2 for the missing caller. Its constructor signature point is the one refuted under replication-ondisk-2.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent proposed JSON5, relaxed JSON syntax or a new JSON text type. [P1] (jsonb_translate) and [P6] (pluggable JSON output) are in the same area, but their recorded reasons concern a niche value-replacement function and output callback machinery, and neither reason carries over here. Message #2 cites Andrew's comment in another thread about the incremental parser, and v2 drops incremental parser support in response, so no earlier objection is left open. Message #1 links the configuration format proposal as the origin, but the material does not report any objection from it.
- standards-compat: 3 objection(s). Not applicable. The implicit json to json5 cast and its effect on function resolution could not be checked without the catalog entries.
- replication-ondisk: 2 objection(s). The other persistent changes are the new pg_type, pg_proc and pg_cast rows, storage as verbatim text, and a binary-coercible json to json5 cast. These add no WAL, wire protocol or relation format change, so they raise no concern from this angle.
- security: 0 objection(s). The proposal adds no privileges, roles or file access, and every existing caller, including the libpq OAuth and manifest code, passes json5=false, so untrusted input does not reach the new grammar there. Parse depth and input size limits are unchanged, so we see no new denial-of-service path without reading the code.
- maintenance-fit: 3 objection(s). Not applicable; three objections listed. Hot-path cost for strict JSON was measured at about 1% noise (message #2), so I raise no performance objection.
- already-exists: 2 objection(s). Two objections raised. The JSONC-only first commit also lacks a named in-core consumer, so it is covered by the second objection.
- demand: 2 objection(s). Two objections raised. Both are 'not now, show demand' rather than rejection of the idea.

### Precedents retrieved

- P1: [Add jsonb_translate(jsonb, from, to)](https://commitfest.postgresql.org/patch/6086/) (cf, Withdrawn, 2026-01-25) - used in the review; reason summarised from its thread
- P2: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant; reason summarised from its thread
- P3: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P4: [Revert "Add gitignore entries for jsonpath_gram.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b084b38310cfe9c8b58cc615a81df625c771f5d) (revert, Reverted, 2019-03-22) - judged not relevant
- P5: [psql JSON output format](https://commitfest.postgresql.org/patch/4707/) (cf, Returned with feedback, 2024-05-15) - judged not relevant; reason summarised from its thread
- P6: [Generalized JSON output](https://commitfest.postgresql.org/patch/254/) (cf, Returned with feedback, 2015-07-14) - used in the review; reason summarised from its thread
- P7: [json_table function PLAN clause](https://commitfest.postgresql.org/patch/6332/) (cf, Withdrawn, 2026-05-19) - judged not relevant; reason summarised from its thread
- P8: [Optimize JsonbContainerTypeName by reordering type checks](https://commitfest.postgresql.org/patch/6000/) (cf, Withdrawn, 2026-01-04) - judged not relevant; reason summarised from its thread
- P9: [Removing unused trailing linepointers from heap pages](https://commitfest.postgresql.org/patch/3035/) (cf, Returned with feedback, 2021-12-02) - judged not relevant; reason summarised from its thread
- P10: [\[WIP\] Show a human-readable n_distinct in pg_stats view](https://commitfest.postgresql.org/patch/2061/) (cf, Withdrawn, 2019-08-28) - judged not relevant
- P11: [SQL/JSON support in PostgreSQL](https://commitfest.postgresql.org/patch/1063/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P12: [Relax requirement for INTO with SELECT in pl/pgsql](https://commitfest.postgresql.org/patch/586/) (cf, Returned with feedback, 2016-09-22) - judged not relevant
- P13: [GnuTLS support](https://commitfest.postgresql.org/patch/1277/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P14: [unnesting multirange data types](https://commitfest.postgresql.org/patch/3178/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P15: [seccomp-bpf support](https://commitfest.postgresql.org/patch/2263/) (cf, Rejected, 2020-01-07) - judged not relevant
- P16: [Sample configuration files](https://commitfest.postgresql.org/patch/745/) (cf, Returned with feedback, 2016-09-30) - judged not relevant
- P17: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P18: [support for MODULE](https://commitfest.postgresql.org/patch/3528/) (cf, Returned with feedback, 2022-07-18) - judged not relevant
- P19: [conversion from size in human readable text format to text](https://commitfest.postgresql.org/patch/432/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P20: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P21: [Support Int64 GUCs](https://commitfest.postgresql.org/patch/5253/) (cf, Rejected, 2024-09-29) - judged not relevant
- P22: [Add SQL syntax check](https://commitfest.postgresql.org/patch/4704/) (cf, Rejected, 2024-05-15) - judged not relevant
- P23: [Revert "Remove hard coded formats for INT64 and use configured settings instead."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=348c10efe08f01872daeeb62f32c8e362fcbba69) (revert, Reverted, 2011-04-27) - judged not relevant
- P24: [Configurable file mode mask](https://commitfest.postgresql.org/patch/1069/) (cf, Returned with feedback, 2017-03-21) - judged not relevant

### Warnings

- attachment limit (8) reached; 0001-Add-JSON5-comment-support-to-the-JSON-lexer.patch not read
- attachment limit (8) reached; 0004-Add-JSON5-single-quoted-string-support-to-the-JSON-l.patch not read
- attachment limit (8) reached; 0005-Add-JSON5-number-extensions-to-the-JSON-lexer.patch not read
- attachment limit (8) reached; 0002-Add-JSON5-trailing-comma-support-to-the-JSON-parser.patch not read
- attachment limit (8) reached; 0003-Add-JSON5-unquoted-object-key-support-to-the-JSON-pa.patch not read
- attachment limit (8) reached; 0006-Add-JSON5-multi-line-string-support-to-the-JSON-lexe.patch not read
- attachment limit (8) reached; 0008-Add-json5-type-documentation.patch not read
- attachment limit (8) reached; 0007-Add-json5-data-type-with-cast-paths-to-json-jsonb.patch not read

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 146 s; cost reported by the CLI: $1.82
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 111 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:15:49.681Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
