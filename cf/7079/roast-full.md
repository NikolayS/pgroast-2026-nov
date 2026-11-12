# Design pre-review: New data type binary format documentation

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7079](https://commitfest.postgresql.org/patch/7079/) (Needs review); authors: Mark Wong (markwkm)
- Thread: [real/float example for testlibpq3](https://www.postgresql.org/message-id/flat/Yhf70G9OYIKueYZi%40workstation-mark-wong), started 2022-02-24
- Messages read: 25 of 25
- Patch prose read: 0001-doc-Add-binary-data-format-descriptions.patch, testlibpq3-real-v1.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal, and the review notes record that Tom Lane and Robert Haas asked for prose documentation of the binary formats. Two fixable design points remain: the patch removes testlibpq3 without a stated reason (maintenance-fit-2), and the chapter introduction does not say whether layouts are stable across versions (standards-compat-2). Confidence is medium because the chapter hunk is truncated and nobody has replied to the July 2026 patch.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The chapter hunk is truncated after the first numeric types, so this review could not confirm coverage of all built-in types or the accuracy of any layout. It also could not confirm whether a later section carries a cross-version note.

## The proposal, restated

**Problem.** Driver authors who use the binary protocol must read each type's send and receive functions to learn its layout, because the docs only say integers are big-endian. This affects libpq users and drivers in every language.

**Proposed design.** The patch adds a new chapter, "Binary Format of Data Types", under Client Interfaces, with general conventions and a prose layout for each built-in type, ordered like the Data Types chapter. It links to the new chapter from datatype.sgml and protocol.sgml. It also removes testlibpq3 and libpq Example Program 3 from the docs.

**User-visible surface.**
- New documentation chapter "Binary Format of Data Types" (binary-format.sgml) in the Client Interfaces part.
- New index terms "binary format" and "wire format".
- libpq Example Program 3 (testlibpq3.c, testlibpq3.sql) is removed from the docs and from src/test/examples.

**What the author claims.**
- The chapter covers every built-in data type, written in prose without code examples, as Tom Lane and Robert Haas asked.
- The server's send and receive functions stay authoritative if the docs and the source disagree.
- Mark Wong says he wrote the patch with help from Claude.

**Thread state.** Only Mark Wong's 2026-07-30 patch message is visible after the 2023 Returned with Feedback; nobody has replied to it. Earlier, Tom Lane, Robert Haas and Peter Eisentraut agreed on prose docs in a separate chapter, but Peter preferred the Data Types chapter.

## 2. Prior attempts

No precedent found (corpus searched for items before 2022-02-24; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Removing testlibpq3 drops the only built example of binary parameters

fixable · level 2, rubric item 12 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A libpq user looks for a complete program that passes out-of-line or binary parameters through PQexecParams and finds no Example Program 3. Tutorials that link to the libpq-example-3 anchor now get a broken link. The example lived in src/test/examples, where it can still be compiled; the prose that replaces it cannot.

The removed program's own header says it tests "out-of-line parameters and binary I/O", so it covers more than type layouts. In message #10 the plan was to leave testlibpq3.c as it was, and no reviewer asked for its removal. The commit message gives no reason for the removal. Keeping the example costs little, and removing it is unrelated to documenting formats.

Thread: [#10 (Mark Wong, 2022-06-16)](https://www.postgresql.org/message-id/Yqu1MmYGrOQLsD7D%40workstation-mark-wong), [#25 (Mark Wong, 2026-07-30)](https://www.postgresql.org/message-id/amvWEvqYLT8F6Rhh%40localhost)

**Question:** Why must testlibpq3 go in the same patch, and what libpq example then shows binary parameters and out-of-line parameters with PQexecParams?

Counter-check: held. The commit message and file list confirm testlibpq3 is removed, and its header says it also shows out-of-line parameters with PQexecParams. No reviewer asked for removal and the patch gives no reason; keeping the example would fix this.

### 2. Chapter does not say whether binary layouts are stable across versions

fixable · level 2, rubric item 6 · design, wire protocol, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A driver author builds a binary decoder from the version 20 chapter and assumes it covers every server the driver supports. A later release changes one type's send function, and the driver either breaks or the change is blocked as a documented contract break. Nobody notices until users report wrong values.

protocol.sgml keeps its warning that complex binary representations may change across server versions, but the new chapter presents each layout as fact with no versioning note. Published per-type prose is likely to be read by driver authors as a contract, which is stronger than today's source-code-only status in message #5. The chapter says the source wins on disagreement, but says nothing on change policy or history.

Thread: [#5 (Tom Lane, 2022-02-28)](https://www.postgresql.org/message-id/2746817.1646088530%40sss.pgh.pa.us), [#25 (Mark Wong, 2026-07-30)](https://www.postgresql.org/message-id/amvWEvqYLT8F6Rhh%40localhost)

**Question:** Does the chapter promise binary layout stability across major versions, and where does it tell readers how a layout change will be recorded?

Counter-check: held. The visible chapter introduction has no cross-version caveat, and readers arriving from datatype.sgml never see the one kept in protocol.sgml. The chapter hunk is truncated, so a later note cannot be excluded; one sentence would fix it.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | The new chapter documents the binary wire format of every built-in type, and the patch edits protocol.sgml to link to it. |
| catalog changes | no |  |
| ABI / extension API | **yes** | see objection maintenance-fit-2: Removing testlibpq3 drops the only built example of binary parameters |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A chapter of about 970 lines must follow every later change to a send or receive function and every new built-in type. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why does the patch remove testlibpq3, and which libpq example then shows PQexecParams with out-of-line and binary parameters?
2. Does the chapter promise that binary layouts are stable across major versions, and where does it tell readers how a layout change is recorded?
3. Does the chapter cover every built-in type, including composite, range and multirange types, and how was each layout checked against its send function?
4. Why does the chapter sit under Client Interfaces when Peter Eisentraut preferred the Data Types chapter?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner would be a committer who knows the type send and receive functions and the frontend/backend protocol. Tom Lane took part earlier in the thread, but no committer has replied to the July 2026 patch.
- **Liability:** The committer signs for the accuracy of about 970 lines of layout prose, where each error can become a driver bug. The chapter then needs an update whenever a binary format changes or a built-in type is added.
- **Author follow-up:** The author came back with a full rewrite after the 2023 Returned with Feedback and moved the entry to the next CommitFest in October 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Direct readers are driver authors, a small group whose work reaches most users. The patch is docs-only, but a reviewer has to compare each type's prose with its send function, and both remaining objections have small fixes.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Removing testlibpq3 drops the only PQexecParams binary example. Refuted (not specific): Duplicate of maintenance-fit-2, which states the same concern more accurately. Example Program 3 is an anchor in the libpq examples section, so readers lose a link target and an example, but no page goes missing.
- `maintenance-fit-1` Nothing keeps the new chapter in sync with send and receive functions. Refuted (not specific): No reference documentation in Postgres is checked against the code, including the text formats in datatype.sgml, so the chapter follows existing practice. A send function change already breaks drivers today, and messages #9 and #20 asked for this documentation.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and the earlier returns in this thread (messages #17, #22, #24) were for inactivity. The patch follows the direction agreed in messages #9, #15, #19 and #20: prose only, all built-in types, a new chapter under Client Interfaces. Removing testlibpq3 goes against message #10, where the plan was to leave it unchanged, but that belongs to another angle.
- standards-compat: 2 objection(s). No SQL syntax, GUC, catalog or pg_dump change is involved; the patch is documentation plus removal of one example program.
- replication-ondisk: 0 objection(s). The patch only adds documentation and removes an example program, so it changes no on-disk format, WAL, catalog, protocol message or ABI. protocol.sgml keeps the warning that binary formats may change across server versions, so the new chapter does not freeze the wire format. Keeping the chapter in sync with send and receive functions is a maintenance question for another angle.
- security: 0 objection(s). The patch only changes documentation and removes a test program, so it adds no privilege or exposure. Binary send and receive formats are already public in the source, and no server behaviour, default, role or access path changes. The removed testlibpq3 example sets a secure search_path, but losing that example teaches less and creates no attack path.
- maintenance-fit: 2 objection(s). The patch changes only documentation, so the core idea has no hot-path, ABI or extension-fit cost. Tom Lane and Robert Haas asked for this chapter.
- already-exists: 0 objection(s). The only existing way is reading each type's send and receive functions in the server source, and Tom Lane (message #5, #9) and Robert Haas (message #8, #20) asked for prose docs to replace exactly that. I know of no contrib module or external tool that documents these formats, so this angle finds nothing in core or outside it that already meets the need.
- demand: 0 objection(s). The thread shows real demand. In message #4 Chapman Flack points out that every binary-protocol driver needs these layouts, and in messages #5, #9 and #20 Tom Lane and Robert Haas say real documentation would be better than reading the source. The cost is a docs-only chapter, and the removal of testlibpq3 and keeping the chapter in sync belong to other angles.

### Precedents retrieved

- P1: [Add uri percent-encoding for binary data](https://commitfest.postgresql.org/patch/2299/) (cf, Returned with feedback, 2020-07-29) - judged not relevant; reason summarised from its thread
- P2: [Binary I/O for isn extension](https://commitfest.postgresql.org/patch/639/) (cf, Returned with feedback, 2016-09-28) - judged not relevant; reason summarised from its thread
- P3: [Revert binary change in copyright year adjustment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=1358801b7a7afed840b780d7c52e20921a483ed1) (revert, Reverted, 2012-01-02) - judged not relevant
- P4: [get rid of StdRdOptions, use individual binary reloptions representation for each relation kind instead](https://commitfest.postgresql.org/patch/1536/) (cf, Returned with feedback, 2019-09-27) - judged not relevant
- P5: [_bt_binsrch* improvements - equal-prefix-skip binary search](https://commitfest.postgresql.org/patch/2731/) (cf, Withdrawn, 2020-09-15) - judged not relevant; reason summarised from its thread
- P6: [Revert binary change in copyright year adjustment.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b5eb06a22a5a811cbdd065ac52580bbcc59469a0) (revert, Reverted, 2012-01-02) - judged not relevant
- P7: [guc-ify the formerly hard-coded MAX_SEND_SIZE to max_wal_send](https://commitfest.postgresql.org/patch/958/) (cf, Returned with feedback, 2017-03-28) - judged not relevant; reason summarised from its thread
- P8: [Revert: looks like Binary Large OBject\[sic\] wasn't a misspelling](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7b243aa666e49431f45bec71565d8ca49b7c9ef6) (revert, Reverted, 2010-08-17) - judged not relevant
- P9: [psql export import bytea data](https://commitfest.postgresql.org/patch/913/) (cf, Rejected, 2017-09-07) - judged not relevant; reason summarised from its thread
- P10: [Escaping from blocked send() by pg_terminate_backend().](https://commitfest.postgresql.org/patch/84/) (cf, Rejected, 2015-01-12) - judged not relevant; reason summarised from its thread
- P11: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - judged not relevant; reason summarised from its thread
- P12: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P13: [Sending WAL receiver feedback regularly even if the receiver is under heavy load](https://commitfest.postgresql.org/patch/143/) (cf, Returned with feedback, 2015-05-16) - judged not relevant
- P14: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P15: [Hooks and API to allow loadable wire protocol extensions](https://commitfest.postgresql.org/patch/3018/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P16: [protocol default result formats setting](https://commitfest.postgresql.org/patch/2812/) (cf, Returned with feedback, 2021-04-07) - judged not relevant
- P17: [Built-in connection proxy](https://commitfest.postgresql.org/patch/2067/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P18: [Fix documentation for COPY protocol flow](https://commitfest.postgresql.org/patch/3175/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P19: [Built-in connection pooler](https://commitfest.postgresql.org/patch/3043/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [Support ORDER BY in CREATE FUNCTION for Set Returning Functions](https://commitfest.postgresql.org/patch/88/) (cf, Rejected, 2015-05-03) - judged not relevant
- P22: [Extended protocol COPY FROM STDIN fix](https://commitfest.postgresql.org/patch/3180/) (cf, Returned with feedback, 2021-12-01) - judged not relevant
- P23: [add --if-exists to pg_recvlogical](https://commitfest.postgresql.org/patch/1256/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P24: [Application name for pg_basebackup, pg_receivewal and pg_recvlogical](https://commitfest.postgresql.org/patch/2335/) (cf, Rejected, 2019-11-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 18; wall time: 116 s; cost reported by the CLI: $2.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 2 from cache, 6 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 63 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:18:04.138Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
