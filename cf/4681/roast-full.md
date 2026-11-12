# Design pre-review: Make COPY format extendable: Extract COPY TO format implementations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4681](https://commitfest.postgresql.org/patch/4681/) (Needs review); authors: Kouhei Sutou (kou)
- Thread: [Make COPY format extendable: Extract COPY TO format implementations](https://www.postgresql.org/message-id/flat/20231204.153548.2126325458835528809.kou%40clear-code.com), started 2023-12-04
- Messages read: 341 of 341
- Patch prose read: v1-0001-Extract-COPY-TO-format-implementations.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the problem is stated and demand is shown across several years. Four fixable level 2 objections survived. maintenance-fit-2 (options accepted with no effect) and maintenance-fit-3 (no in-tree format that moves bytes) are the strongest, followed by replication-ondisk-1, replication-ondisk-3 and maintenance-fit-4. These are design points a committer who knows COPY can settle; none requires a different proposal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, and only 3 of 341 messages were visible directly. The per-row cost for built-in formats, the exact option checks in v3 and the format codes sent to the client could not be checked.

## The proposal, restated

**Problem.** COPY TO/FROM supports only text, csv and binary, and users have asked for JSON lines, Apache Arrow, Avro, Parquet and ORC. The author wants such formats to live in extensions, because they depend on external libraries and may not be written in C. The author also wants faster custom formats than conversions through text.

**Proposed design.** Built-in formats already go through CopyToRoutine/CopyFromRoutine callbacks, committed earlier. The current v3 set (Sawada) moves the COPY state structs into a public header and adds a C registration API that an extension calls in _PG_init(). It also adds an opaque per-format state pointer, a per-option callback, a whole-options validation callback and a test module.

**User-visible surface.**
- COPY option FORMAT accepts a format name registered by a loaded extension.
- C API: RegisterCopyCustomFormat() with CopyToRoutine and CopyFromRoutine, called from _PG_init().
- New header copy_state.h exposing CopyFromStateData and CopyToStateData, with an opaque format_private pointer.
- Callbacks for format options: one per option, one to validate all options after processing.
- Proposed exports of CopySendData(), CopySendEndOfRow() and CopyGetData() (reviewer diff, not yet in v3).

**What the author claims.**
- The refactoring keeps behavior and tests unchanged.
- The first benchmark showed no significant loss for text, csv and binary COPY TO at 100K to 10M rows.
- Unqualified text, csv and binary always use the built-in formats, which keeps pg_dump output compatible.
- Registration through _PG_init() avoids search_path problems and is secure by design.
- Later benchmarks by Tomas Vondra and Sutou showed differences mostly within 5% with no reproducible trend.

**Thread state.** Design moved from SQL handler functions to _PG_init() registration with support from Paquier and Sawada; Andres Freund raised performance concerns that no one answered conclusively. Two reviews of v3 (Manu, Joao Rito) found that extensions cannot move bytes without exporting the I/O functions, the test module fails on current master, and v3 needs a rebase.

## 2. Prior attempts

### Adding Support for Copy callback functionality on COPY TO api (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/2660/](https://commitfest.postgresql.org/patch/2660/)
- **Thread:** [https://www.postgresql.org/message-id/flat/253C21D1-FCEB-41D9-A2AF-E6517015B7D7@amazon.com](https://www.postgresql.org/message-id/flat/253C21D1-FCEB-41D9-A2AF-E6517015B7D7@amazon.com)
- **Date:** decided 2020-09-30; started 2020-07-01
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author did not answer review feedback for two weeks, so the patch was returned with feedback. The open review points were to run pgindent, to add an in-tree user of the callback, and to have the callback return the number of bytes written and report failure.
- **Reason (quote):** "This feedback has not been answered after two weeks, so I have marked the patch as returned with feedback." ([message](https://www.postgresql.org/message-id/20200930074151.GR1996%40paquier.xyz))
- **What is different this time:** P3 added only a destination callback for COPY TO. This proposal registers whole formats in both directions. P3 ended because review comments went unanswered, which does not apply to this thread.


## 3. Objections a reviewer is likely to raise

### 1. Every new COPY option needs a decision in every custom format

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), ABI / extension API · `maintenance-fit-2`

A user runs COPY with FORMAT ndjson, ON_ERROR ignore, and DELIMITER '|'. Both options are accepted, have no effect, and a bad row still aborts with "line 0". When core later adds another option like reject_limit, each custom format accepts it silently until its author notices.

Message #341 shows DELIMITER, HEADER, NULL and FORCE_QUOTE accepted and ignored, and ON_ERROR accepted but not applied. The built-in checks are skipped for custom formats, so core has no way to reject options a format does not support. Each future COPY option then becomes a silent gap in every format until someone notices.

Thread: [#341 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com)

**Question:** What rule makes core reject a COPY option that a custom format does not declare support for, including ON_ERROR and future options?

Counter-check: held. Message #341 shows DELIMITER, HEADER, NULL and FORCE_QUOTE accepted with no effect, and ON_ERROR ignore accepted while a bad row still fails. v3 leaves rejection to each format's validation callback, and the proposal states no default rule in core.

### 2. No in-tree format that moves bytes to guard the API

fixable · level 2, rubric item 11 · ABI / extension API, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

A committer changes how CopyGetData or CopySendEndOfRow handles the frontend or PROGRAM destination. The in-tree test module only raises NOTICEs, so it still passes. Out-of-tree formats break, and the project learns about it from bug reports after release.

Message #340 found that v3 exports no I/O functions, so the test module cannot write or read data, and the module fails on master. Message #341 confirms both points. A NOTICE-only module cannot show that real formats keep working, which is the concern recorded in [P8]. [P3] was also asked for an in-tree user of a new COPY callback.

**Precedents.**
- [ctidscan as an example of custom-scan](https://commitfest.postgresql.org/patch/18/) (Rejected, 2015-07-22); thread: [link](https://www.postgresql.org/message-id/flat/9A28C8860F777E439AA12E8AEA7694F801091310@BPXM15GP.gisp.nec.co.jp)
- [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (Returned with feedback, 2020-09-30); thread: [link](https://www.postgresql.org/message-id/flat/253C21D1-FCEB-41D9-A2AF-E6517015B7D7@amazon.com)

Thread: [#340 (Manu, 2026-09-22)](https://www.postgresql.org/message-id/179011382359.2771650.8713483121447030047%40gmail.com), [#341 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com)

**Question:** Will the series include an in-tree format that reads and writes real rows through the exported functions on file, frontend and PROGRAM paths?

Counter-check: held. Messages #340 and #341 show the in-tree module only raises NOTICEs, which hid the missing I/O exports. [P3] supports the request for an in-tree user; [P8] concerned the usefulness of the custom-scan interface and must not be used as support.

### 3. Custom format names stored in file_fdw options have no catalog anchor

fixable · level 2, rubric item 14 · pg_dump / restore round-trip, on-disk format / pg_upgrade, catalog changes · `replication-ondisk-1`

A DBA creates a file_fdw foreign table with format 'ndjson' and loads the library only through LOAD. pg_dump emits the table, and on restore or during pg_upgrade's schema restore the file_fdw validator rejects it with 'COPY format "ndjson" not recognized'. A later removal of the library from shared_preload_libraries leaves the table failing at query time.

Message #341 shows that file_fdw accepts a custom format and that COPY fails when the library is not loaded. The format name is then persisted in ftoptions, but registration lives only in backend memory, so pg_depend and pg_dump know nothing about it. The proposal does not say how restore, pg_upgrade or library removal should behave for objects that name such a format.

Thread: [#341 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com), [#271 (Masahiko Sawada, 2025-05-01)](https://www.postgresql.org/message-id/CAD21AoBuEqcz2_%2BdpA3WTiDUF%3DFgudPBKwM%2BnvH%2BqHT-k4p5mA%40mail.gmail.com)

**Question:** How does a file_fdw table using a custom format survive pg_dump, pg_restore and pg_upgrade when the library is not preloaded during the restore?

Counter-check: held. Message #341 confirms both halves: file_fdw stores a custom format name, and the name fails in a session without the library. Message #294 accepted the preload requirement for COPY commands, but the thread does not discuss restore of stored options.

### 4. Wire protocol format code for custom formats is not defined

fixable · level 2, rubric item 10 · wire protocol · `replication-ondisk-3`

A client driver runs COPY TO STDOUT with an Arrow format and reads CopyOutResponse. If the server reports the overall format as text (0), a driver that treats text COPY data as character data in client_encoding can mangle the binary stream. The user finds out when the received file does not parse.

CopyOutResponse and CopyInResponse carry an overall format and per-column codes that only distinguish text and binary. Message #8 asked how column-oriented formats would work at the protocol level, and the visible material does not answer it. Message #341 tested \copy only with ndjson, which is text, so a binary format through the frontend path is untested.

Thread: [#8 ("Daniel Verite", 2023-12-06)](https://www.postgresql.org/message-id/ee94f987-f4ca-4a0c-96d7-909465d20808%40manitou-mail.org), [#341 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com)

**Question:** Which overall and per-column format codes does the server send in CopyOutResponse and CopyInResponse for a custom format, and who chooses them?

Counter-check: held. CopyOutResponse and CopyInResponse carry a text or binary code, and the visible v3 material does not say what a custom format sends. Message #10 answered message #8 only about row groups, and message #341 tested the frontend path only with a text format.

### 5. Exported COPY I/O functions may block future COPY speedups

fixable · level 2, rubric item 6 · performance on the hot path, ABI / extension API · `maintenance-fit-4`

A developer wants to rework COPY FROM buffering, for example by parsing directly from raw_buf in batches. Extensions now call CopyGetData and read copy_file directly, as message #341 proposes for Parquet. The rework either breaks those extensions or has to keep the old per-call interface alive.

Message #309 asks about the COPY speedups this design forecloses, separately from the measured overhead. Message #310 answers only about indirection layers, and message #332 says that question is still open. Exporting CopyGetData, CopySendData and copy_file access fixes the current I/O layer as a contract.

Thread: [#309 (Andres Freund, 2025-07-15)](https://www.postgresql.org/message-id/awcj7u3i5nf5yksmcoq24mtodd7kbmxljaygf72txhu6jxcd34%40ckdj2omq7v6u), [#310 (Masahiko Sawada, 2025-07-17)](https://www.postgresql.org/message-id/CAD21AoAQkjU%3Do0nX4y0jtX0BnsrqA04g2ABqrUwjT88YeEWarA%40mail.gmail.com), [#332 (Tomas Vondra, 2025-11-17)](https://www.postgresql.org/message-id/c36d218a-bb38-42b9-9076-cb75b8984a39%40vondra.me), [#340 (Manu, 2026-09-22)](https://www.postgresql.org/message-id/179011382359.2771650.8713483121447030047%40gmail.com), [#341 (Joao Rito, 2026-11-10)](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com)

**Question:** Which planned changes to COPY buffering or I/O would still be possible once CopyGetData, CopySendData and copy_file are part of the extension API?

Counter-check: held. Message #310 answered message #309 only about indirection, and message #332 says the question of which COPY improvements the design blocks is still open. No message says whether a format may read copy_file directly, so the contract for the I/O layer is undefined.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | file_fdw stores a custom format name in its options, and pg_upgrade's schema restore must resolve it. |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | COPY TO STDOUT and FROM STDIN with a custom format send CopyOutResponse and CopyInResponse format codes. |
| catalog changes | **yes** | see objection replication-ondisk-1: Custom format names stored in file_fdw options have no catalog anchor |
| ABI / extension API | **yes** | RegisterCopyCustomFormat(), copy_state.h with both COPY state structs, and proposed exports of the COPY I/O functions. |
| SQL-standard conflict or non-standard syntax | **yes** | The COPY option FORMAT accepts names registered by a loaded extension. |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | **yes** | pg_dump emits file_fdw options that name a custom format, which restores only when the library is loaded. |
| psql UX conventions | **yes** | \copy passes a custom format through the frontend path; message #341 tested it only with a text format. |
| security / privileges | **yes** | Format code loaded through _PG_init() receives the relation and the full values arrays during COPY. |
| backpatchability | **yes** | Fields of the exported COPY state structs constrain what minor-release fixes may change. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every future COPY option and every change to COPY buffering must consider out-of-tree formats. |
| performance on the hot path | **yes** | Built-in text, csv and binary COPY run through the callback routines; benchmarks showed differences mostly within 5%. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. What rule makes core reject a COPY option, including ON_ERROR, DELIMITER and future options, that a custom format does not declare support for?
2. Will the series include an in-tree format that reads and writes real rows through the exported I/O functions on file, frontend and PROGRAM paths?
3. How does a file_fdw table that names a custom format survive pg_dump, pg_restore and pg_upgrade when the library is not preloaded?
4. Which overall and per-column format codes does the server send in CopyOutResponse and CopyInResponse for a custom format, and who chooses them?
5. May a COPY FROM format read copy_file directly, and which changes to COPY buffering stay possible once CopyGetData and CopySendData are exported?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Sawada writes the current v3 series and Paquier supported the _PG_init() design. The owner needs to know the COPY code and extension API practice.
- **Liability:** The owner signs for a permanent extension API: the registration function, the exposed COPY state structs, the exported I/O functions and the option rules. Later COPY changes must keep out-of-tree formats working.
- **Author follow-up:** The author has moved the entry through 13 CommitFests since 2023 and replies in the thread. v3 needs a rebase, and the reviews in messages #340 and #341 have no visible answer yet.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Demand for JSON lines, Arrow and Parquet has lasted since 2018 and a committer writes the series. The open points are the option rule, a real in-tree format, the I/O contract and the protocol format code, across a 341-message thread.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Only a NOTICE-only in-tree module checks the custom format API. Refuted (not specific): Duplicate of maintenance-fit-3, which makes the same point and also cites [P3], where reviewers asked for an in-tree user of a COPY callback.
- `standards-compat-1` Core text/CSV options are silently accepted with a custom format. Refuted (not specific): Duplicate of maintenance-fit-2, which covers the same silently accepted text/CSV options and adds ON_ERROR and future options.
- `standards-compat-2` Format names share one global namespace with future core formats. Refuted (already handled): Messages #279, #284, #286 and #294 weighed this collision and chose a flat name space where built-in names win and duplicates raise an error. Custom EXPLAIN options and logical decoding plugins follow the same rule, and the cost falls on users of the extension.
- `standards-compat-3` Objects that name a custom format do not restore unless the library is preloaded. Refuted (not specific): Duplicate of replication-ondisk-1. Its claim that CREATE EXTENSION never makes the format available is also too strong, because creating a C function loads the library in that session.
- `replication-ondisk-2` Exposing the full COPY state structs freezes their layout as ABI. Refuted (already handled): Messages #321, #327 and #336 weighed full exposure and chose it to match existing practice such as table AM scan descriptors. The rule against adding fields mid-struct in a minor release already applies to every installed header, as the objection itself notes.
- `security-1` Core does not keep custom formats inside the COPY column list. Refuted (no failure scenario): The scenario needs a format that ignores attnumlist, and the only module in the material honours it (message #340). Postgres trusts loaded C code in the same way for FDWs and table AMs, so this is an extension bug that core does not guard against elsewhere.
- `maintenance-fit-1` Whole COPY state structs become extension ABI with no boundary. Refuted (no failure scenario): The scenario cannot happen as written. A renamed field fails at compile time, the module magic check stops a library built for another major version from loading, and minor releases do not reorder struct fields. Message #336 dropped the split on purpose.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent rejected extensible COPY formats as an idea. The COPY RAW rows [P11], [P14] and [P16] add a built-in format and record no reason, so they were not used. Message #1 cites earlier threads that it says reached consensus on extensibility, conditional on showing no performance loss.
- standards-compat: 3 objection(s)
- replication-ondisk: 3 objection(s). The proposal adds no WAL records, on-disk relation format or catalog. Logical replication table sync and pg_dump data use unqualified text, which always resolves to the built-in format.
- security: 1 objection(s). Loading through _PG_init() keeps format code under administrator control. Built-in names cannot be shadowed, and core still checks file and program privileges, so I see no other exposure.
- maintenance-fit: 4 objection(s). Owner signal is good: Sawada, a committer, writes the current series, and Paquier supported the _PG_init design. The per-row cost claim is plausible because v3 changes only option processing.
- already-exists: 0 objection(s). Existing tools do not cover the main use case. An FDW such as parquet_fdw (message #10) or COPY (SELECT row_to_json(...)) can handle server-side files or JSON output, but neither can stream Arrow or Parquet through COPY FROM STDIN or TO STDOUT over the frontend protocol. Committers have supported moving formats into extensions since 2018 (messages #1, #294), so this angle finds no objection.
- demand: 0 objection(s). The material shows lasting, independent demand: requests for JSON lines, Arrow, Avro, Parquet and ORC from 2018, 2022 and 2023 (message #1), with working extensions for Arrow and JSON lines and a user-side Parquet request (message #341). Committers are engaged, and an existing FDW workaround (message #10) does not cover COPY's role in loading and unloading data.

### Precedents retrieved

- P1: [Allow COPY "text" to output a header and add header matching mode to COPY FROM](https://commitfest.postgresql.org/patch/2504/) (cf, Returned with feedback, 2020-09-29) - judged not relevant; reason summarised from its thread
- P2: [Allow COPY's 'text' format to output a header](https://commitfest.postgresql.org/patch/1629/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P3: [Adding Support for Copy callback functionality on COPY TO api](https://commitfest.postgresql.org/patch/2660/) (cf, Returned with feedback, 2020-09-30) - used in the review; reason summarised from its thread
- P4: [Custom signals and handlers for extension](https://commitfest.postgresql.org/patch/1430/) (cf, Returned with feedback, 2020-11-26) - judged not relevant; reason summarised from its thread
- P5: [csv output format for psql](https://commitfest.postgresql.org/patch/1500/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P6: [copy_srf (was COPY as a set returning function)](https://commitfest.postgresql.org/patch/869/) (cf, Returned with feedback, 2017-01-31) - judged not relevant; reason summarised from its thread
- P7: [Fix formatting of Interval output](https://commitfest.postgresql.org/patch/3548/) (cf, Withdrawn, 2022-02-19) - judged not relevant; reason summarised from its thread
- P8: [ctidscan as an example of custom-scan](https://commitfest.postgresql.org/patch/18/) (cf, Rejected, 2015-07-22) - used in the review; reason summarised from its thread
- P9: [Custom compression methods](https://commitfest.postgresql.org/patch/1294/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P10: [Formatting of warning about using ident](https://commitfest.postgresql.org/patch/2155/) (cf, Withdrawn, 2019-07-24) - judged not relevant
- P11: [COPY RAW](https://commitfest.postgresql.org/patch/223/) (cf, Returned with feedback, 2016-04-08) - used in the review
- P12: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant
- P13: [Revert removal of signed, volatile, and signal handler arg type tests.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ef6164de1d60f65039a73f5975d50b4fa10b8a5b) (revert, Reverted, 2000-08-29) - judged not relevant
- P14: [COPY RAW](https://commitfest.postgresql.org/patch/547/) (cf, Returned with feedback, 2016-03-18) - used in the review
- P15: [psql markdown output format](https://commitfest.postgresql.org/patch/1056/) (cf, Returned with feedback, 2017-11-28) - judged not relevant
- P16: [RAW COPY](https://commitfest.postgresql.org/patch/676/) (cf, Returned with feedback, 2016-12-15) - used in the review
- P17: [Throw an error from to_char() instead of printing a string of "#" when a number doesn't fit in the desired output format.](https://wiki.postgresql.org/wiki/Not_Worth_Doing#Character_Formatting) (do_not_want, Not worth doing, date unknown) - judged not relevant
- P18: [Inconsistency in startup process's MyBackendId and procsignal array registration with ProcSignalInit()](https://commitfest.postgresql.org/patch/3357/) (cf, Withdrawn, 2022-02-27) - judged not relevant
- P19: [Revert "Change SHA2 implementation based on OpenSSL to use EVP digest routines"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fe0a1dc52c7332a65b44db8e8408a5fd1d8fc8fb) (revert, Reverted, 2020-09-29) - judged not relevant
- P20: [Pluggable toaster](https://commitfest.postgresql.org/patch/3490/) (cf, Rejected, 2023-04-03) - judged not relevant
- P21: [Revert the addition of GetMaxBackends() and related stuff.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7fc0e7de9fb8306e84d1c15211aba4308f694455) (revert, Reverted, 2022-04-12) - judged not relevant
- P22: [Jsonb transform for pl/python](https://commitfest.postgresql.org/patch/1336/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P23: [SQL/JSON: JSON_TABLE](https://commitfest.postgresql.org/patch/1473/) (cf, Returned with feedback, 2020-08-03) - judged not relevant
- P24: [Remove page-read callback from XLogReaderState](https://commitfest.postgresql.org/patch/2113/) (cf, Returned with feedback, 2021-10-04) - judged not relevant

### Warnings

- 251 long messages in the middle of the thread were left out of the prompts

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 205 s; cost reported by the CLI: $9.33
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 5 of 24 kept
- Tone check: 110 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:28:43.024Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
