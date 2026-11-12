# Design pre-review: XMLElement() and XMLForest() embed the XML declaration of xml arguments

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7622](https://commitfest.postgresql.org/patch/7622/); [thread](https://www.postgresql.org/message-id/flat/b13f297c-09da-4300-a3b1-6a3fa34accb5%40uni-muenster.de), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection to the idea or the design survived for master: the change follows SQL/XML 10.19, xmlconcat(), Oracle and Db2, and it fixes output that xml_in() rejects. The one surviving objection, standards-compat-1, is fixable and concerns the stable target only. A committer who knows the xml code has to decide whether minor releases should change query output.

Not checked: Without the code, this review cannot check that attribute values and other callers of xml_out_internal() keep their output, or what the new regression tests cover.

## Prior attempts

None found before 2026-11-10.

## Objections a reviewer is likely to raise

### 1. Back-patching changes xmlelement() and query_to_xml() output in stable branches

An application on a minor release stores or compares xmlelement() or query_to_xml() output that contains an embedded declaration, for example to check standalone="yes". After a minor update the same query returns different text without the declaration. The user finds out when a checksum, a diff or a downstream parser that reads the inner declaration gives a different result.

The CommitFest entry targets stable, so the patch changes query results inside a minor release. The current output cannot be read back by xml_in(), which argues for a fix, but the text form is still returned to clients and may be used as text. Message #1 makes the standards case for master but does not say why a minor release should change output. A committer would likely ask whether only master should get the change, with a release note.

**Question:** Why should stable branches change xmlelement(), xmlforest() and query_to_xml() output, rather than only master with a release note?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/b13f297c-09da-4300-a3b1-6a3fa34accb5%40uni-muenster.de); `standards-compat-1`</sub>

## Questions for the author

1. Why should stable branches change xmlelement(), xmlforest() and query_to_xml() output, rather than only master with a release note?
2. How should users find and repair rows already stored with an embedded declaration, which still fail on restore after the patch?
3. Do other functions that embed xml values, such as xmlagg() or xmlroot(), copy the declaration in the same way?
4. Does it matter for the result that the encoding or a version other than 1.0 in the embedded declaration is dropped?
5. Does the new flag change the signature of a function that extensions can call in stable branches?

## Touch points

- **ABI / extension API:** The patch adds a flag to xml_out_internal() in xml.c; the material does not show whether an exported signature changes in stable branches.
- **SQL-standard conflict or non-standard syntax:** The patch changes xmlelement() and xmlforest() output to follow SQL/XML 6.15, 6.16 and 10.19.
- **pg_dump / restore round-trip:** Stored xmlelement() results with an embedded declaration fail on restore; the patch stops new ones but does not repair existing rows.
- **backpatchability:** The CommitFest entry targets stable, so minor releases would change xmlelement(), xmlforest() and query_to_xml() output.

## Who would commit it, and when

- Owner: silent. A committer who knows the xml data type code in xml.c and the back-patch policy would have to own it; none has spoken in the one visible message.
- Priority: proceed; audience niche; review cost low. The bug shows only for xml values with a standalone property or a version other than 1.0, but it produces values that do not restore from a dump. The patch is 5.0 KB, touches xml.c and its regression test, and adds no user-visible surface.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
