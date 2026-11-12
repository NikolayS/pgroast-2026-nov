# Design pre-review: New data type binary format documentation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7079](https://commitfest.postgresql.org/patch/7079/); [thread](https://www.postgresql.org/message-id/flat/Yhf70G9OYIKueYZi%40workstation-mark-wong), 25 of 25 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No surviving objection questions the goal, and the review notes record that Tom Lane and Robert Haas asked for prose documentation of the binary formats. Two fixable design points remain: the patch removes testlibpq3 without a stated reason (maintenance-fit-2), and the chapter introduction does not say whether layouts are stable across versions (standards-compat-2). Confidence is medium because the chapter hunk is truncated and nobody has replied to the July 2026 patch.

Not checked: The chapter hunk is truncated after the first numeric types, so this review could not confirm coverage of all built-in types or the accuracy of any layout. It also could not confirm whether a later section carries a cross-version note.

## Prior attempts

None found before 2022-02-24.

## Objections a reviewer is likely to raise

### 1. Removing testlibpq3 drops the only built example of binary parameters

A libpq user looks for a complete program that passes out-of-line or binary parameters through PQexecParams and finds no Example Program 3. Tutorials that link to the libpq-example-3 anchor now get a broken link. The example lived in src/test/examples, where it can still be compiled; the prose that replaces it cannot.

The removed program's own header says it tests "out-of-line parameters and binary I/O", so it covers more than type layouts. In message #10 the plan was to leave testlibpq3.c as it was, and no reviewer asked for its removal. The commit message gives no reason for the removal. Keeping the example costs little, and removing it is unrelated to documenting formats.

**Question:** Why must testlibpq3 go in the same patch, and what libpq example then shows binary parameters and out-of-line parameters with PQexecParams?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#10](https://www.postgresql.org/message-id/Yqu1MmYGrOQLsD7D%40workstation-mark-wong), [#25](https://www.postgresql.org/message-id/amvWEvqYLT8F6Rhh%40localhost); `maintenance-fit-2`</sub>

### 2. Chapter does not say whether binary layouts are stable across versions

A driver author builds a binary decoder from the version 20 chapter and assumes it covers every server the driver supports. A later release changes one type's send function, and the driver either breaks or the change is blocked as a documented contract break. Nobody notices until users report wrong values.

protocol.sgml keeps its warning that complex binary representations may change across server versions, but the new chapter presents each layout as fact with no versioning note. Published per-type prose is likely to be read by driver authors as a contract, which is stronger than today's source-code-only status in message #5. The chapter says the source wins on disagreement, but says nothing on change policy or history.

**Question:** Does the chapter promise binary layout stability across major versions, and where does it tell readers how a layout change will be recorded?

<sub>fixable, level 2, design; messages [#5](https://www.postgresql.org/message-id/2746817.1646088530%40sss.pgh.pa.us), [#25](https://www.postgresql.org/message-id/amvWEvqYLT8F6Rhh%40localhost); `standards-compat-2`</sub>

## Questions for the author

1. Why does the patch remove testlibpq3, and which libpq example then shows PQexecParams with out-of-line and binary parameters?
2. Does the chapter promise that binary layouts are stable across major versions, and where does it tell readers how a layout change is recorded?
3. Does the chapter cover every built-in type, including composite, range and multirange types, and how was each layout checked against its send function?
4. Why does the chapter sit under Client Interfaces when Peter Eisentraut preferred the Data Types chapter?

## Touch points

- **wire protocol:** The new chapter documents the binary wire format of every built-in type, and the patch edits protocol.sgml to link to it.
- **ABI / extension API:** see objection maintenance-fit-2: Removing testlibpq3 drops the only built example of binary parameters
- **maintenance burden ("who maintains this in 10 years"):** A chapter of about 970 lines must follow every later change to a send or receive function and every new built-in type.

## Who would commit it, and when

- Owner: silent. The owner would be a committer who knows the type send and receive functions and the frontend/backend protocol. Tom Lane took part earlier in the thread, but no committer has replied to the July 2026 patch.
- Priority: proceed; audience niche; review cost medium. Direct readers are driver authors, a small group whose work reaches most users. The patch is docs-only, but a reviewer has to compare each type's prose with its send function, and both remaining objections have small fixes.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
