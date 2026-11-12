# Design pre-review: Add XMLSerialize: version (SQL/XML X076), explicit declaration (SQL/XML X078)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5279](https://commitfest.postgresql.org/patch/5279/); [thread](https://www.postgresql.org/message-id/flat/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), 14 of 14 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the clauses are SQL/XML standard syntax that only core can add. The remaining points are fixable: already-exists-1 asks what the clauses add over xmlroot(), and standards-compat-1, standards-compat-3 and maintenance-fit-2 ask about CONTENT mode output, the warning for '1.1' and per-row checks. Nobody has reviewed the design since message #2, so someone who knows the xml code must settle these.

Not checked: Without the code, the review could not check if the new keywords are reserved, what ruleutils.c emits for views, or where the version check runs. Only 3 of 14 messages were shown, so later discussion may answer some questions.

## Prior attempts

None found before 2024-09-25.

## Objections a reviewer is likely to raise

### 1. Most of the result is reachable today with xmlroot() and string functions

A user who needs a declaration with a chosen version can already run `xmlserialize(document xmlroot(x, version '1.0', standalone no) as text)`, or prepend a literal `<?xml version="1.0"?>` with `||`. To drop a declaration, they can use `regexp_replace` on the serialized text. The patch adds keywords and a second way to set the version that core must keep consistent with `xmlroot`.

Postgres already has `xmlroot(xml, version ..., standalone ...)`, which sets the version in a value's XML declaration before serialization. The thread names no user who needs the new clauses (messages #1-#14), and the only stated motivation is marking X076 and X078 as supported. Standard conformance is a real reason, so this is not fatal, but a committer would likely ask what the clauses give beyond `xmlroot` plus string handling, and whether two version mechanisms validate the version string the same way.

**Question:** Which real use case cannot be met by xmlroot() plus xmlserialize() or simple string functions, and do both paths accept and reject the same version strings?

<sub>fixable, level 1b, desirability; messages [#1](https://www.postgresql.org/message-id/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), [#3](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com); `already-exists-1`</sub>

### 2. CONTENT mode can emit a declaration no XML parser accepts

A user runs xmlserialize(CONTENT '<a/><b/>text'::xml AS text INCLUDING XMLDECLARATION) to feed an external consumer. The output is '<?xml version="1.0"?><a/><b/>text'. That is not a well-formed document, and as an external entity its text declaration lacks the encoding that XML 1.0 requires, so the consumer's parser rejects it at load time.

The v10 doc states that in CONTENT mode the declaration is emitted "even if it would not normally be considered valid by the XML specification". The same patch flips X078 to YES in sql_features.txt, so information_schema.sql_features reports full support. The proposal does not say what SQL/XML requires for CONTENT with INCLUDING XMLDECLARATION, or why emitting malformed output is better than raising an error or documenting a deviation. Nobody discussed this in the thread after message #3.

**Question:** What does SQL/XML specify for CONTENT values that are not well-formed documents under INCLUDING XMLDECLARATION, and does the patch match it?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#3](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#14](https://www.postgresql.org/message-id/20fc4be0-24be-4a7a-a39d-2c33b9701c25%40uni-muenster.de); `standards-compat-1`</sub>

### 3. VERSION '1.1', a value the standard names, raises a WARNING

An application ported from another SQL/XML implementation uses VERSION '1.1' INCLUDING XMLDECLARATION in a bulk export. Every call returns a WARNING built from libxml2's parser message. If the warning fires per row, the client and server log fill with noise, and drivers that surface warnings report a problem for valid standard SQL.

Message #1 quotes the standard: the literal "shall be '1.0' or '1.1'". The patch delegates validity to libxml2, as Tom Lane suggested in message #2. As a result, a standard-sanctioned value is flagged as suspect, and the wording of the warning depends on the libxml2 build. The proposal does not say if the check runs once at parse time on the literal or for each evaluated row.

**Question:** Is the version checked once per statement or once per row, and why should a standard-listed value like '1.1' produce a warning at all?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#1](https://www.postgresql.org/message-id/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), [#2](https://www.postgresql.org/message-id/3712088.1727280167%40sss.pgh.pa.us), [#3](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com); `standards-compat-3`</sub>

### 4. Version check may run, and warn, once per row

A report runs xmlserialize(... VERSION '1.1' INCLUDING XMLDECLARATION) over a million-row table. If the check runs in the executor for each value, the query parses an extra document per row and sends a million WARNING messages to the client and server log.

The files touched include execExprInterp.c and xml.c, and the prose says libxml2 parses a minimal document to validate the version. The prose does not say if this happens once at parse or plan time or for every value. The SQL/XML clause takes a character string literal, so a single check before execution looks possible.

**Question:** Is the VERSION string validated once per statement or once per evaluated value, and how many warnings does a multi-row query emit?

<sub>fixable, level 2, performance on the hot path; messages [#3](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com); `maintenance-fit-2`</sub>

### 5. Replacing a declaration may drop the input's standalone attribute

A stored document starts with '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'. An export job adds VERSION '1.0' INCLUDING XMLDECLARATION. If the rewritten declaration is version-only, standalone="yes" is lost, and a downstream validator now fetches the external DTD and applies its defaults. The user finds out only when downstream results change.

The commit message says the declaration is "added, replaced or removed" and that a written declaration never has an encoding. It does not say what happens to standalone. The encoding omission is justified by xml_out(), but that reason does not cover standalone, which carries document semantics and is not affected by encoding conversion.

**Question:** When VERSION or INCLUDING XMLDECLARATION replaces an existing declaration, is the input's standalone value kept, and is that what the standard requires?

<sub>fixable, level 2, design; messages [#3](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de); `standards-compat-2`</sub>

## Questions for the author

1. What does SQL/XML require for INCLUDING XMLDECLARATION on a CONTENT value that is not a well-formed document, and does the patch match it?
2. What do the new clauses give that xmlroot() plus xmlserialize() does not, and do both paths accept and reject the same version strings?
3. Is the VERSION string checked once per statement or once per evaluated value, and how many warnings does a multi-row query emit?
4. Why does VERSION '1.1', a value the standard names, raise a warning, and why did v10 start checking the version in CONTENT mode?
5. When a declaration is replaced, is the input's standalone value kept, and how do the options combine with INDENT?

## Touch points

- **catalog changes:** sql_features.txt marks X076 and X078 as supported, and new XmlExpr fields change stored view rules.
- **ABI / extension API:** The patch changes node definitions in primnodes.h and parsenodes.h, which extensions that inspect XmlExpr see.
- **SQL-standard conflict or non-standard syntax:** The patch adds the SQL/XML clauses VERSION and INCLUDING/EXCLUDING XMLDECLARATION, with new keywords in kwlist.h.
- **pg_dump / restore round-trip:** ruleutils.c must deparse the new clauses so that views using them dump and restore; the output is not shown.
- **maintenance burden ("who maintains this in 10 years"):** The error or warning for a version string comes from libxml2, and xmlroot() already sets the version by another path.
- **performance on the hot path:** The patch touches execExprInterp.c, and the libxml2 version check may run for each evaluated value.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** xmlroot() plus xmlserialize() already sets a declaration version, so part of the result exists today.

## Who would commit it, and when

- Owner: silent. A committer who knows xml.c, the libxml2 integration and the SQL/XML grammar would have to own it. Tom Lane answered one validation question in message #2 and has not reviewed the patch.
- Priority: needs champion; audience niche; review cost medium. The patch adds two standard SQL/XML features for a small group of XML users, and its cost falls mostly on those users. It has waited through 11 CommitFests with a tester but no design review, so it needs a committer who knows the xml code.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
