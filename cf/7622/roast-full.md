# Design pre-review: XMLElement() and XMLForest() embed the XML declaration of xml arguments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7622](https://commitfest.postgresql.org/patch/7622/) (Needs review); authors: Jim Jones (jimus)
- Thread: [XMLElement() and XMLForest() embed the XML declaration of xml arguments](https://www.postgresql.org/message-id/flat/b13f297c-09da-4300-a3b1-6a3fa34accb5%40uni-muenster.de), started 2026-11-10
- Messages read: 1 of 1
- Patch prose read: v1-0001-Omit-the-XML-declaration-of-xml-values-embedded-a.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection to the idea or the design survived for master: the change follows SQL/XML 10.19, xmlconcat(), Oracle and Db2, and it fixes output that xml_in() rejects. The one surviving objection, standards-compat-1, is fixable and concerns the stable target only. A committer who knows the xml code has to decide whether minor releases should change query output.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review cannot check that attribute values and other callers of xml_out_internal() keep their output, or what the new regression tests cover.

## The proposal, restated

**Problem.** When an xml value with an XML declaration is used as element content, xmlelement() and xmlforest() copy the declaration into the element. A declaration is only allowed at the start of a document, so xml_in() rejects the result and a dump containing it cannot be restored.

**Proposed design.** The patch adds a flag to xml_out_internal() to omit the declaration. map_sql_value_to_xml_value() uses this flag when it embeds an xml value as element content, which covers xmlelement(), xmlforest() and the query_to_xml() family. Attribute values are not affected.

**User-visible surface.**
- Output of xmlelement() changes: the declaration of an xml argument used as content is no longer copied into the element.
- Output of xmlforest() changes in the same way.
- Output of the query_to_xml() family changes in the same way.

**What the author claims.**
- SQL/XML (ISO/IEC 9075-14, sections 6.15, 6.16 and 10.19) puts only the children of the document node into the element, so the declaration is dropped.
- Oracle and Db2 also drop the declaration.
- xmlconcat() already drops the declarations of its arguments.
- The bug only shows when xml_out() would print the declaration, i.e. for a standalone property or a version other than 1.0.
- The fix is a bugfix suitable for stable branches, per the CommitFest target version.

**Thread state.** Only the first message is visible. The author asks for thoughts on the approach.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-11-10; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Back-patching changes xmlelement() and query_to_xml() output in stable branches

fixable · level 2, rubric item 6 · backpatchability, SQL-standard conflict or non-standard syntax · `standards-compat-1`

An application on a minor release stores or compares xmlelement() or query_to_xml() output that contains an embedded declaration, for example to check standalone="yes". After a minor update the same query returns different text without the declaration. The user finds out when a checksum, a diff or a downstream parser that reads the inner declaration gives a different result.

The CommitFest entry targets stable, so the patch changes query results inside a minor release. The current output cannot be read back by xml_in(), which argues for a fix, but the text form is still returned to clients and may be used as text. Message #1 makes the standards case for master but does not say why a minor release should change output. A committer would likely ask whether only master should get the change, with a release note.

Thread: [#1 (Jim Jones, 2026-11-10)](https://www.postgresql.org/message-id/b13f297c-09da-4300-a3b1-6a3fa34accb5%40uni-muenster.de)

**Question:** Why should stable branches change xmlelement(), xmlforest() and query_to_xml() output, rather than only master with a release note?

Counter-check: held. The CommitFest entry targets stable, and message #1 makes the standards case without weighing a result change in a minor release. The old output fails xml_in(), which favours a fix, but clients still receive it as text, so the question stands.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds a flag to xml_out_internal() in xml.c; the material does not show whether an exported signature changes in stable branches. |
| SQL-standard conflict or non-standard syntax | **yes** | The patch changes xmlelement() and xmlforest() output to follow SQL/XML 6.15, 6.16 and 10.19. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | Stored xmlelement() results with an embedded declaration fail on restore; the patch stops new ones but does not repair existing rows. |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, so minor releases would change xmlelement(), xmlforest() and query_to_xml() output. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why should stable branches change xmlelement(), xmlforest() and query_to_xml() output, rather than only master with a release note?
2. How should users find and repair rows already stored with an embedded declaration, which still fail on restore after the patch?
3. Do other functions that embed xml values, such as xmlagg() or xmlroot(), copy the declaration in the same way?
4. Does it matter for the result that the encoding or a version other than 1.0 in the embedded declaration is dropped?
5. Does the new flag change the signature of a function that extensions can call in stable branches?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the xml data type code in xml.c and the back-patch policy would have to own it; none has spoken in the one visible message.
- **Liability:** The committer signs for a change of query output in every branch that gets the patch, and for reports from users whose stored or compared text differs after a minor update.
- **Author follow-up:** The author asks for thoughts on the approach in message #1; no other sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The bug shows only for xml values with a standalone property or a version other than 1.0, but it produces values that do not restore from a dump. The patch is 5.0 KB, touches xml.c and its regression test, and adds no user-visible surface.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Back-patching changes xmlelement() output in a minor release. Refuted (not specific): Duplicate of standards-compat-1, which states the same back-patch concern with the same failure scenario and a more concrete request to the author.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P6] is about how xmlconcat() handles processing instructions and DTDs, a different function and question, and it has no recorded reason that could apply to dropping declarations in xmlelement() or xmlforest(). Message #1 points to no earlier discussion of this idea, so the prior-attempts angle has nothing to raise.
- standards-compat: 1 objection(s). The change matches SQL/XML 10.19, xmlconcat() and two other databases, so I have no standards objection to it on master. The only objection concerns back-patching. [P6] is about xmlconcat() and has no recorded reason, so this review does not rely on it.
- replication-ondisk: 0 objection(s). The patch changes only the text that xmlelement(), xmlforest() and the query_to_xml() family produce, with no change to WAL, catalogs, the wire protocol or extension ABI. Rows already stored with an embedded declaration stay as they are and still fail on restore. The patch does not make that worse, but the release note could tell users how to find and fix such rows.
- security: 0 objection(s). The patch changes only the serialized output of xmlelement(), xmlforest() and the query_to_xml() family. It adds no privilege, role, setting or exposure path, and it uses no more resources than the current code. Dropping the declaration makes values easier to restore and does not let a user bypass row-level security or column privileges. Without the code, a reader cannot check whether the new flag also affects error messages.
- maintenance-fit: 1 objection(s). From the maintenance and fit angle the design is small and local to xml.c. It adds no hot-path cost for queries without xml, follows existing xmlconcat() behaviour, and belongs in core. [P6] has no recorded reason and covers a different xmlconcat() change, so it is not cited.
- already-exists: 0 objection(s). No existing feature gives the same result: xmlelement(), xmlforest() and query_to_xml() produce output that xml_in() rejects, and no extension can fix core output that cannot be restored. [P6] covers PIs and DTDs in xmlconcat(), records no reason, and does not apply to dropping declarations in element content.
- demand: 0 objection(s). This is a correctness bug: xmlelement() can store a value that xml_in() rejects, so a dump with it does not restore. Such a bug needs no demand evidence, and standard text, Oracle, Db2 and xmlconcat() agree on the fix; [P6] has no recorded reason. Whether to back-patch with no field report is left to other angles.

### Precedents retrieved

- P1: [Declarative partitioning](https://commitfest.postgresql.org/patch/331/) (cf, Returned with feedback, 2015-12-02) - judged not relevant; reason summarised from its thread
- P2: [Declarative partitioning](https://commitfest.postgresql.org/patch/469/) (cf, Returned with feedback, 2016-03-21) - judged not relevant; reason summarised from its thread
- P3: [ECPG: Declare Statement](https://commitfest.postgresql.org/patch/936/) (cf, Returned with feedback, 2017-11-16) - judged not relevant; reason summarised from its thread
- P4: [Revert "Add DECLARE STATEMENT support to ECPG."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96b6c82c9dd4a6a91c7e54bf42d36da111959ec6) (revert, Reverted, 2019-09-20) - judged not relevant
- P5: [Added support of XML_PARSE_HUGE flag for XML documents](https://commitfest.postgresql.org/patch/4772/) (cf, Rejected, 2024-01-31) - judged not relevant; reason summarised from its thread
- P6: [Improve handling of PIs and DTDs in xmlconcat() \[5\]](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - used in the review
- P7: [Allow XML arrays to be cast to other data types](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P8: [Restructure XML and /contrib/xml2 functionality](https://wiki.postgresql.org/wiki/Not_Worth_Doing#XML) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P9: [Add XMLCast (SQL/XML X025)](https://commitfest.postgresql.org/patch/5110/) (cf, Withdrawn, 2026-09-08) - judged not relevant; reason summarised from its thread
- P10: [Propagate sanity checks of ProcessUtility() to standard_ProcessUtility()?](https://commitfest.postgresql.org/patch/4877/) (cf, Withdrawn, 2024-05-18) - judged not relevant; reason summarised from its thread
- P11: [plpgsql - possibility to get element or array type for referenced variable type](https://commitfest.postgresql.org/patch/412/) (cf, Rejected, 2016-03-16) - judged not relevant; reason summarised from its thread
- P12: [Relax transactional restrictions on ALTER TYPE ... ADD VALUE (redux).](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=212fab9926b2f0f04b0187568e7124b70e8deee5) (revert, Partially reverted, 2018-10-08) - judged not relevant
- P13: [Extending the range operators so they can cope with range <op> element and element <op> range in addition to the existing range <op> range](https://commitfest.postgresql.org/patch/2747/) (cf, Returned with feedback, 2021-04-08) - judged not relevant; reason summarised from its thread
- P14: [introduce optimized linear search functions that return index of matching element](https://commitfest.postgresql.org/patch/3906/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P15: [Implement ASSERTIONs per the SQL standard.](https://commitfest.postgresql.org/patch/1604/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P16: [pg_upgrade fails with non-standard ACL](https://commitfest.postgresql.org/patch/2215/) (cf, Returned with feedback, 2021-11-26) - judged not relevant
- P17: [Revert ill-considered change to plpgsql:](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4627a8f41975886ddfe4e482d1453cec797415bb) (revert, Reverted, 2006-05-27) - judged not relevant
- P18: [Allow meson to find ICU in non-standard location](https://commitfest.postgresql.org/patch/4203/) (cf, Withdrawn, 2023-03-08) - judged not relevant
- P19: [Revert "Replace pg_restrict by standard restrict"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fa16e7fd84886643f480c36614fa11d45fc6d26f) (revert, Reverted, 2026-01-14) - judged not relevant
- P20: [Extending with C: doc on how to call internal functions](https://commitfest.postgresql.org/patch/5259/) (cf, Withdrawn, 2024-09-18) - judged not relevant
- P21: [Embedded server](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Features_We_Do_Not_Want) (do_not_want, Not wanted, date unknown) - judged not relevant
- P22: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P23: [Add option --drop-cascade for pg_dump/restore](https://commitfest.postgresql.org/patch/3065/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P24: [xml2: add test for coverage](https://commitfest.postgresql.org/patch/3833/) (cf, Returned with feedback, 2023-01-31) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 91 s; cost reported by the CLI: $0.80
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 56 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T01:09:26.225Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
