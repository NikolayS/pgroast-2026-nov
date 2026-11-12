# Design pre-review: Add XMLSerialize: version (SQL/XML X076), explicit declaration (SQL/XML X078)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5279](https://commitfest.postgresql.org/patch/5279/) (Needs review); authors: Jim Jones (jimus)
- Thread: [XMLSerialize: version and explicit XML declaration](https://www.postgresql.org/message-id/flat/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), started 2024-09-25
- Messages read: 14 of 14
- Patch prose read: v10-0001-Add-XMLSerialize-version-and-explicit-XML-declar.patch, v1-0001-Add-XMLSerialize-version-and-explicit-XML-declara.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the clauses are SQL/XML standard syntax that only core can add. The remaining points are fixable: already-exists-1 asks what the clauses add over xmlroot(), and standards-compat-1, standards-compat-3 and maintenance-fit-2 ask about CONTENT mode output, the warning for '1.1' and per-row checks. Nobody has reviewed the design since message #2, so someone who knows the xml code must settle these.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check if the new keywords are reserved, what ruleutils.c emits for views, or where the version check runs. Only 3 of 14 messages were shown, so later discussion may answer some questions.

## The proposal, restated

**Problem.** XMLSERIALIZE gives no way to add, remove or set the version of the XML declaration in its output. The patch adds SQL/XML features X076 (VERSION) and X078 (explicit XML declaration), which Postgres lists as not supported. The thread does not name users who need them.

**Proposed design.** The patch adds VERSION, INCLUDING XMLDECLARATION and EXCLUDING XMLDECLARATION to XMLSERIALIZE. With neither declaration option, the output keeps a declaration only if the input had one, and VERSION alone adds none. Without INDENT, only the declaration is added, replaced or removed, never with an encoding, and libxml2 checks the version string in both DOCUMENT and CONTENT mode.

**User-visible surface.**
- New XMLSERIALIZE clause: VERSION <xmlversion>.
- New XMLSERIALIZE clause: INCLUDING XMLDECLARATION | EXCLUDING XMLDECLARATION.
- New keywords in kwlist.h, so a change to the keyword list.
- Error for versions libxml2 rejects, such as '2.0', and a warning for versions it does not fully support, such as '1.1'.
- sql_features.txt marks X076 and X078 as supported, which shows in information_schema.sql_features.

**What the author claims.**
- The behavior without the new options is the same as before.
- Leaving out the encoding declaration matches what xml_out() does, because the result goes through encoding conversion.
- Tom Lane suggested letting libxml2 decide if a version string is valid, and the patch follows this.
- Oracle accepts any version string and Db2 accepts only '1.0', so the patch sits between the two.

**Thread state.** Tom Lane answered the first design question about validation. Since then the author has posted only rebases, and solai v tested v9 and found no problems, but nobody has reviewed the code or the design.

## 2. Prior attempts

No precedent found (corpus searched for items before 2024-09-25; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Most of the result is reachable today with xmlroot() and string functions

fixable · level 1b, rubric item 3 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), SQL-standard conflict or non-standard syntax, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A user who needs a declaration with a chosen version can already run `xmlserialize(document xmlroot(x, version '1.0', standalone no) as text)`, or prepend a literal `<?xml version="1.0"?>` with `||`. To drop a declaration, they can use `regexp_replace` on the serialized text. The patch adds keywords and a second way to set the version that core must keep consistent with `xmlroot`.

Postgres already has `xmlroot(xml, version ..., standalone ...)`, which sets the version in a value's XML declaration before serialization. The thread names no user who needs the new clauses (messages #1-#14), and the only stated motivation is marking X076 and X078 as supported. Standard conformance is a real reason, so this is not fatal, but a committer would likely ask what the clauses give beyond `xmlroot` plus string handling, and whether two version mechanisms validate the version string the same way.

Thread: [#1 (Jim Jones, 2024-09-25)](https://www.postgresql.org/message-id/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), [#3 (Jim Jones, 2024-09-30)](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12 (solai v, 2026-08-04)](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com)

**Question:** Which real use case cannot be met by xmlroot() plus xmlserialize() or simple string functions, and do both paths accept and reject the same version strings?

Counter-check: held. xmlroot() already sets or clears the version in a value's declaration before serialization, so the alternative can do most of the job. The thread does not compare the two paths or say if they validate version strings the same way.

### 2. CONTENT mode can emit a declaration no XML parser accepts

fixable · level 2, rubric item 12 · SQL-standard conflict or non-standard syntax · `standards-compat-1`

A user runs xmlserialize(CONTENT '<a/><b/>text'::xml AS text INCLUDING XMLDECLARATION) to feed an external consumer. The output is '<?xml version="1.0"?><a/><b/>text'. That is not a well-formed document, and as an external entity its text declaration lacks the encoding that XML 1.0 requires, so the consumer's parser rejects it at load time.

The v10 doc states that in CONTENT mode the declaration is emitted "even if it would not normally be considered valid by the XML specification". The same patch flips X078 to YES in sql_features.txt, so information_schema.sql_features reports full support. The proposal does not say what SQL/XML requires for CONTENT with INCLUDING XMLDECLARATION, or why emitting malformed output is better than raising an error or documenting a deviation. Nobody discussed this in the thread after message #3.

Thread: [#3 (Jim Jones, 2024-09-30)](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#14 (Jim Jones, 2026-10-18)](https://www.postgresql.org/message-id/20fc4be0-24be-4a7a-a39d-2c33b9701c25%40uni-muenster.de)

**Question:** What does SQL/XML specify for CONTENT values that are not well-formed documents under INCLUDING XMLDECLARATION, and does the patch match it?

Counter-check: held. The v10 doc itself says CONTENT mode may emit a declaration the XML specification does not allow, while X078 becomes YES. Postgres already accepts such values, which lowers the severity, but nobody in the thread says what SQL/XML requires here.

### 3. VERSION '1.1', a value the standard names, raises a WARNING

fixable · level 2, rubric item 10 · SQL-standard conflict or non-standard syntax · `standards-compat-3`

An application ported from another SQL/XML implementation uses VERSION '1.1' INCLUDING XMLDECLARATION in a bulk export. Every call returns a WARNING built from libxml2's parser message. If the warning fires per row, the client and server log fill with noise, and drivers that surface warnings report a problem for valid standard SQL.

Message #1 quotes the standard: the literal "shall be '1.0' or '1.1'". The patch delegates validity to libxml2, as Tom Lane suggested in message #2. As a result, a standard-sanctioned value is flagged as suspect, and the wording of the warning depends on the libxml2 build. The proposal does not say if the check runs once at parse time on the literal or for each evaluated row.

Thread: [#1 (Jim Jones, 2024-09-25)](https://www.postgresql.org/message-id/0427cced-93df-4e50-aaa0-645438a2dbe4%40uni-muenster.de), [#2 (Tom Lane, 2024-09-25)](https://www.postgresql.org/message-id/3712088.1727280167%40sss.pgh.pa.us), [#3 (Jim Jones, 2024-09-30)](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12 (solai v, 2026-08-04)](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com)

**Question:** Is the version checked once per statement or once per row, and why should a standard-listed value like '1.1' produce a warning at all?

Counter-check: held. Tom Lane's reply in message #2 covers who decides validity, but it does not cover a user-visible warning for '1.1', which the standard names. The per-row part belongs to maintenance-fit-2.

### 4. Version check may run, and warn, once per row

fixable · level 2, rubric item 8 · performance on the hot path · `maintenance-fit-2`

A report runs xmlserialize(... VERSION '1.1' INCLUDING XMLDECLARATION) over a million-row table. If the check runs in the executor for each value, the query parses an extra document per row and sends a million WARNING messages to the client and server log.

The files touched include execExprInterp.c and xml.c, and the prose says libxml2 parses a minimal document to validate the version. The prose does not say if this happens once at parse or plan time or for every value. The SQL/XML clause takes a character string literal, so a single check before execution looks possible.

Thread: [#3 (Jim Jones, 2024-09-30)](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de), [#12 (solai v, 2026-08-04)](https://www.postgresql.org/message-id/CAF0whuehdj8xrVLmcUs%3D5HYMx-SYDsV%2B8WxXXE%2BCb9ApUSMAWw%40mail.gmail.com)

**Question:** Is the VERSION string validated once per statement or once per evaluated value, and how many warnings does a multi-row query emit?

Counter-check: held. The patch touches execExprInterp.c and the examples show the warning at run time. The prose does not say if the check runs once per statement or once per value.

### 5. Replacing a declaration may drop the input's standalone attribute

fixable · level 2, rubric item 12 · design, SQL-standard conflict or non-standard syntax · `standards-compat-2`

A stored document starts with '<?xml version="1.0" encoding="UTF-8" standalone="yes"?>'. An export job adds VERSION '1.0' INCLUDING XMLDECLARATION. If the rewritten declaration is version-only, standalone="yes" is lost, and a downstream validator now fetches the external DTD and applies its defaults. The user finds out only when downstream results change.

The commit message says the declaration is "added, replaced or removed" and that a written declaration never has an encoding. It does not say what happens to standalone. The encoding omission is justified by xml_out(), but that reason does not cover standalone, which carries document semantics and is not affected by encoding conversion.

Thread: [#3 (Jim Jones, 2024-09-30)](https://www.postgresql.org/message-id/765f1f4a-ab9b-4fb1-bcbb-9ae77ca534f0%40uni-muenster.de)

**Question:** When VERSION or INCLUDING XMLDECLARATION replaces an existing declaration, is the input's standalone value kept, and is that what the standard requires?

Counter-check: held. The commit message and doc explain the missing encoding but do not mention standalone. The scenario depends on behavior the prose does not state, so it stands as a question the patch must answer.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | **yes** | sql_features.txt marks X076 and X078 as supported, and new XmlExpr fields change stored view rules. |
| ABI / extension API | **yes** | The patch changes node definitions in primnodes.h and parsenodes.h, which extensions that inspect XmlExpr see. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch adds the SQL/XML clauses VERSION and INCLUDING/EXCLUDING XMLDECLARATION, with new keywords in kwlist.h. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | ruleutils.c must deparse the new clauses so that views using them dump and restore; the output is not shown. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | The error or warning for a version string comes from libxml2, and xmlroot() already sets the version by another path. |
| performance on the hot path | **yes** | The patch touches execExprInterp.c, and the libxml2 version check may run for each evaluated value. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | xmlroot() plus xmlserialize() already sets a declaration version, so part of the result exists today. |

## 5. Questions the author must answer before anyone reviews code

1. What does SQL/XML require for INCLUDING XMLDECLARATION on a CONTENT value that is not a well-formed document, and does the patch match it?
2. What do the new clauses give that xmlroot() plus xmlserialize() does not, and do both paths accept and reject the same version strings?
3. Is the VERSION string checked once per statement or once per evaluated value, and how many warnings does a multi-row query emit?
4. Why does VERSION '1.1', a value the standard names, raise a warning, and why did v10 start checking the version in CONTENT mode?
5. When a declaration is replaced, is the input's standalone value kept, and how do the options combine with INDENT?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows xml.c, the libxml2 integration and the SQL/XML grammar would have to own it. Tom Lane answered one validation question in message #2 and has not reviewed the patch.
- **Liability:** The owner signs for new keywords, the deparse of the new clauses in stored views, and user-visible errors and warnings that libxml2 produces, for as long as XMLSERIALIZE exists.
- **Author follow-up:** The author has rebased the patch through 11 CommitFests up to v10, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** The patch adds two standard SQL/XML features for a small group of XML users, and its cost falls mostly on those users. It has waited through 11 CommitFests with a tester but no design review, so it needs a committer who knows the xml code.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Accepted VERSION strings depend on the installed libxml2 release. Refuted (already handled): In message #2 a committer proposed libxml2 as the authority on version strings, as Postgres already does for xml input. The scenario of a future libxml2 release changing how it treats '1.1' is speculation with no support in the material.
- `demand-1` No user who needs these options is named in two years. Refuted (already handled): The material shows evidence the rubric counts as demand: two SQL/XML standard features that Oracle and Db2 implement, per message #1. The remaining question, what the clauses add over existing functions, is covered by already-exists-1.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at XMLSERIALIZE VERSION or XML declaration options. The thread cites no earlier discussion of this idea, so this angle gives no objection.
- standards-compat: 3 objection(s). Keyword reservation and ruleutils.c deparse (pg_dump of views) could not be judged from the prose. Both are listed as unclear, and no failure can be shown without the code.
- replication-ondisk: 0 objection(s). The patch adds no persistent or cross-node state: no WAL, no on-disk format change, no wire protocol change and no catalog columns. Its only stored effect is new XmlExpr fields in stored view rules, which need the usual catversion bump; ruleutils.c deparse covers pg_dump, though its output is not shown.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: the new options only change the text of the caller's own XMLSERIALIZE result, with no new data access, catalog state, or code run by other roles. Libxml2 checks the version string, so the brief gives no input that could escape the declaration.
- maintenance-fit: 2 objection(s). The feature is small, standard-defined (X076, X078) and limited to XMLSERIALIZE, so non-users pay almost nothing. No committer has reviewed the design since message #2, which affects committability but is not an objection to the idea.
- already-exists: 1 objection(s). One objection. The new clauses are SQL-standard XMLSERIALIZE syntax, and no extension can add grammar, so core placement is justified if the features are wanted.
- demand: 1 objection(s). Demand is thin but standards conformance and Oracle/Db2 parity give some motivation. Costs fall mostly on feature users, so this is a 'not now, show demand' point, not a rejection.

### Precedents retrieved

- P1: [Declarative partitioning](https://commitfest.postgresql.org/patch/331/) (cf, Returned with feedback, 2015-12-02) - judged not relevant; reason summarised from its thread
- P2: [Added support of XML_PARSE_HUGE flag for XML documents](https://commitfest.postgresql.org/patch/4772/) (cf, Rejected, 2024-01-31) - judged not relevant; reason summarised from its thread
- P3: [Declarative partitioning](https://commitfest.postgresql.org/patch/469/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P4: [ECPG: Declare Statement](https://commitfest.postgresql.org/patch/936/) (cf, Returned with feedback, 2017-11-16) - judged not relevant; reason summarised from its thread
- P5: [Revert "Add DECLARE STATEMENT support to ECPG."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96b6c82c9dd4a6a91c7e54bf42d36da111959ec6) (revert, Reverted, 2019-09-20) - judged not relevant
- P6: [Restructure XML and /contrib/xml2 functionality](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P7: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Improve handling of PIs and DTDs in xmlconcat() \[5\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [SERIALIZABLE on standby servers](https://commitfest.postgresql.org/patch/1799/) (cf, Withdrawn, 2019-07-01) - judged not relevant; reason summarised from its thread
- P10: [Allow reliable XML operation non-UTF8 server encodings (xpath(), in particular, is known to not work)](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P11: [Add code indentation check to cirrus-ci](https://commitfest.postgresql.org/patch/4691/) (cf, Withdrawn, 2024-01-24) - judged not relevant; reason summarised from its thread
- P12: [Revert "Optimize various aggregate deserialization functions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4f3b56eea23554e1756a26080db273156f23f4f2) (revert, Reverted, 2023-10-10) - judged not relevant
- P13: [Revert SQL/JSON features](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f2b18bd3f554e96a8cc885b177211be12288e4a) (revert, Reverted, 2022-09-01) - judged not relevant
- P14: [Add llvm version into the version string](https://commitfest.postgresql.org/patch/5202/) (cf, Returned with feedback, 2024-09-24) - judged not relevant; reason summarised from its thread
- P15: [Create shorthand for including all extra tests](https://commitfest.postgresql.org/patch/4547/) (cf, Rejected, 2024-01-20) - judged not relevant; reason summarised from its thread
- P16: [Add --include-table-data-where option to pg_dump, to export only a subset of table data](https://commitfest.postgresql.org/patch/1644/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P17: [Package version in PG_VERSION and version()](https://commitfest.postgresql.org/patch/1418/) (cf, Rejected, 2018-01-19) - judged not relevant
- P18: [SQL/JSON: functions](https://commitfest.postgresql.org/patch/1472/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P19: [Can we include capturing logs of pgdata/pg_upgrade_output.d/*/log in buildfarm](https://commitfest.postgresql.org/patch/4850/) (cf, Rejected, 2024-02-26) - judged not relevant
- P20: [\d+ tablename, \d+ indexname about subpartition](https://commitfest.postgresql.org/patch/4398/) (cf, Returned with feedback, 2024-01-27) - judged not relevant
- P21: [autovectorize page checksum code included elsewhere](https://commitfest.postgresql.org/patch/4649/) (cf, Withdrawn, 2024-04-12) - judged not relevant
- P22: [New \[relation\] options engine](https://commitfest.postgresql.org/patch/3536/) (cf, Returned with feedback, 2023-01-31) - judged not relevant
- P23: [Incremental Materialized View Maintenance](https://commitfest.postgresql.org/patch/2138/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P24: [CALL optional in PL/pgSQL](https://commitfest.postgresql.org/patch/1596/) (cf, Rejected, 2018-04-05) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 158 s; cost reported by the CLI: $2.09
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 82 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:31:26.005Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
