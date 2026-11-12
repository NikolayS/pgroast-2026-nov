# Design pre-review: Proposal: JSON5 support in the JSON parsers

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7021](https://commitfest.postgresql.org/patch/7021/); [thread](https://www.postgresql.org/message-id/flat/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), 2 of 2 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low). The proposal fails at level 1b on demand. The only named consumer is the author's configuration format prototype, and the thread does not say if that proposal still needs this work (demand-1, already-exists-2). An extension can build the json5 type once the parser flag exists (already-exists-1). These are answerable, but the thread has only the author's two messages, so the verdict rests on absence of evidence.

Not checked: The effect of the implicit json to json5 cast on function and operator resolution could not be checked without the catalog entries. The level 2 objections on the type-versus-function choice, encoding-dependent validity and libpq size survived but were not evaluated here.

## Prior attempts

None found before 2026-07-14.

## Objections a reviewer is likely to raise

### 1. No user demand shown for a json5 SQL type

The project adds json5 as a core type with permanent catalog entries and casts, but few users store JSON5 in tables. The type cannot be removed later without breaking dumps and upgrades. Maintainers then carry an ECMAScript 5.1 grammar in the backend for a small group.

The only use case in message #1 is the author's configuration format prototype, and message #2 adds no users. Nobody else has posted, and the listed reviewer has not either. The thread names no -general or -bugs requests and no other databases with a JSON5 column type. A committer would likely ask, as in [P1], why a niche syntax cannot first live in an extension and prove its demand there.

Precedent: [Add jsonb_translate(jsonb, from, to)](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com) (Withdrawn, 2026-01-25)

**Question:** Which users or applications store hand-written JSON5 in Postgres today, and what do they do now, for example strip comments in the client before insert?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com); `demand-1`</sub>

### 2. Clients can turn JSON5 into JSON before it reaches the server

An application that keeps hand-written JSON5 files converts them to JSON with a JSON5 library on the client and inserts the result into json or jsonb. That path works today on every supported release. The in-core grammar adds lexer paths in src/common and about 70 KB to libpq, and no frontend caller is shown to use them.

The only consumer named in message #1 is the configuration format prototype, and the thread does not say whether that proposal still depends on this work. JSON5 parsers already exist outside Postgres, and message #2 tests against the reference one. Extension authors who want JSON5 can bundle such a parser. Core placement is mainly a convenience unless a core caller, such as a server configuration file, needs it.

**Question:** Which in-core caller, backend or frontend, needs to parse JSON5 and cannot convert the input to JSON first?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com); `already-exists-2`</sub>

### 3. The json5 type can be built as an extension on today's hooks

Core ships json5 as a built-in type with no functions or operators. Users who store configuration in it must cast to json or jsonb in every query. The project then keeps the type OID, the casts and the docs through every release and pg_upgrade, even if the configuration format proposal goes another way.

Patch 0007 adds an input function, verbatim storage, a binary-coercible cast from json, and normalizing casts to json and jsonb. An extension can already do all of this with create type, create cast ... without function and create cast with a function. For types outside the built-in OID range, to_json and json_build_object already use a cast to json if one exists. Message #1 says the type commit can be dropped, so the core need left is the parser flag.

Precedent: [Add jsonb_translate(jsonb, from, to)](https://www.postgresql.org/message-id/flat/CA+v5N41RzrpHVSkeN9EOey0JZbhfS3LB4OsRUD7bzTxrbBAszA@mail.gmail.com) (Withdrawn, 2026-01-25)

**Question:** What does a built-in json5 type give users that an extension type, using the patched parser and the existing cast-to-json lookup, cannot?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/CAN4CZFP5dvO4sij414YYG%2B4Hghf3M5m2oPiyR_vNgRj3vhQ_0Q%40mail.gmail.com), [#2](https://www.postgresql.org/message-id/CAN4CZFMmapQG7yJZwjxKhZ%3DzjzDrEK2GYCg%3DFo2_47UYCrkUPA%40mail.gmail.com); `already-exists-1`</sub>

## Questions for the author

1. Does the configuration format proposal still depend on this work, and if so, on which of the eight commits?
2. Which in-core caller, backend or frontend, other than the json5 type, would set the json5 flag?
3. Which users store hand-written JSON5 in Postgres today, and what do they do now instead?
4. What does a built-in json5 type give users that an extension type using the patched parser and the existing cast-to-json lookup cannot?
5. If no frontend caller enables json5, can the json5 code and unicode_category stay out of the libpq build?

## Touch points

- **on-disk format / pg_upgrade:** The json5 type stores validated text verbatim, and that stored form must stay valid across pg_upgrade.
- **catalog changes:** v2-0007 adds rows to pg_type, pg_proc and pg_cast, including an implicit json to json5 cast.
- **ABI / extension API:** makeJsonLexContextCstringLen gets a new json5 parameter and JsonLexContext a new flag.
- **SQL-standard conflict or non-standard syntax:** json5 is a non-standard type name and input syntax next to the SQL/JSON types.
- **pg_dump / restore round-trip:** Raw non-ASCII keys and whitespace are valid only in UTF8 databases, so a dump can fail to restore elsewhere.
- **maintenance burden ("who maintains this in 10 years"):** The shared parser in src/common/jsonapi.c carries a second grammar, and libpq grows by about 70 KB.
- **performance on the hot path:** The patches change the JSON lexer used by all json and jsonb input; message #2 reports about 1% noise.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** An extension could provide the json5 type, and clients can convert JSON5 to JSON today.

## Who would commit it, and when

- Owner: silent. A committer who maintains the common JSON parser (src/common/jsonapi.c) and the json/jsonb types would have to own it; none has posted in this thread.
- Not now, show demand first: A committed or agreed in-core consumer of the json5 flag, such as the configuration format, or requests from independent users who store JSON5 in Postgres. A statement from a JSON parser maintainer that the flag is wanted for extensions would also help.
- Priority: needs design rethink; audience unclear; review cost high. Eight patches of about 150 KiB change the shared lexer and add a permanent type, while the thread names one consumer and has no reviewer reply. Splitting the parser flag from the type and tying the flag to a named consumer would give reviewers something smaller to decide.

---

[roast-full.md](roast-full.md) has 3 more objection(s) that held, 6 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 8.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
