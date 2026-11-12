# Design pre-review: Make COPY format extendable: Extract COPY TO format implementations

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4681](https://commitfest.postgresql.org/patch/4681/); [thread](https://www.postgresql.org/message-id/flat/20231204.153548.2126325458835528809.kou%40clear-code.com), 341 of 341 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the problem is stated and demand is shown across several years. Four fixable level 2 objections survived. maintenance-fit-2 (options accepted with no effect) and maintenance-fit-3 (no in-tree format that moves bytes) are the strongest, followed by replication-ondisk-1, replication-ondisk-3 and maintenance-fit-4. These are design points a committer who knows COPY can settle; none requires a different proposal.

Not checked: The code was not read, and only 3 of 341 messages were visible directly. The per-row cost for built-in formats, the exact option checks in v3 and the format codes sent to the client could not be checked.

## Prior attempts

- [Adding Support for Copy callback functionality on COPY TO api](https://www.postgresql.org/message-id/flat/253C21D1-FCEB-41D9-A2AF-E6517015B7D7@amazon.com) (similar idea; Returned with feedback, 2020-09-30). "This feedback has not been answered after two weeks, so I have marked the patch as returned with feedback." ([message](https://www.postgresql.org/message-id/20200930074151.GR1996%40paquier.xyz)) Different now: P3 added only a destination callback for COPY TO. This proposal registers whole formats in both directions. P3 ended because review comments went unanswered, which does not apply to this thread.

## Objections a reviewer is likely to raise

### 1. Every new COPY option needs a decision in every custom format

A user runs COPY with FORMAT ndjson, ON_ERROR ignore, and DELIMITER '|'. Both options are accepted, have no effect, and a bad row still aborts with "line 0". When core later adds another option like reject_limit, each custom format accepts it silently until its author notices.

Message #341 shows DELIMITER, HEADER, NULL and FORCE_QUOTE accepted and ignored, and ON_ERROR accepted but not applied. The built-in checks are skipped for custom formats, so core has no way to reject options a format does not support. Each future COPY option then becomes a silent gap in every format until someone notices.

**Question:** What rule makes core reject a COPY option that a custom format does not declare support for, including ON_ERROR and future options?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#341](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com); `maintenance-fit-2`</sub>

### 2. No in-tree format that moves bytes to guard the API

A committer changes how CopyGetData or CopySendEndOfRow handles the frontend or PROGRAM destination. The in-tree test module only raises NOTICEs, so it still passes. Out-of-tree formats break, and the project learns about it from bug reports after release.

Message #340 found that v3 exports no I/O functions, so the test module cannot write or read data, and the module fails on master. Message #341 confirms both points. A NOTICE-only module cannot show that real formats keep working, which is the concern recorded in [P8]. [P3] was also asked for an in-tree user of a new COPY callback.

Precedent: [ctidscan as an example of custom-scan](https://www.postgresql.org/message-id/flat/9A28C8860F777E439AA12E8AEA7694F801091310@BPXM15GP.gisp.nec.co.jp) (Rejected, 2015-07-22)

Precedent: [Adding Support for Copy callback functionality on COPY TO api](https://www.postgresql.org/message-id/flat/253C21D1-FCEB-41D9-A2AF-E6517015B7D7@amazon.com) (Returned with feedback, 2020-09-30)

**Question:** Will the series include an in-tree format that reads and writes real rows through the exported functions on file, frontend and PROGRAM paths?

<sub>fixable, level 2, ABI / extension API; messages [#340](https://www.postgresql.org/message-id/179011382359.2771650.8713483121447030047%40gmail.com), [#341](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com); `maintenance-fit-3`</sub>

### 3. Custom format names stored in file_fdw options have no catalog anchor

A DBA creates a file_fdw foreign table with format 'ndjson' and loads the library only through LOAD. pg_dump emits the table, and on restore or during pg_upgrade's schema restore the file_fdw validator rejects it with 'COPY format "ndjson" not recognized'. A later removal of the library from shared_preload_libraries leaves the table failing at query time.

Message #341 shows that file_fdw accepts a custom format and that COPY fails when the library is not loaded. The format name is then persisted in ftoptions, but registration lives only in backend memory, so pg_depend and pg_dump know nothing about it. The proposal does not say how restore, pg_upgrade or library removal should behave for objects that name such a format.

**Question:** How does a file_fdw table using a custom format survive pg_dump, pg_restore and pg_upgrade when the library is not preloaded during the restore?

<sub>fixable, level 2, pg_dump / restore round-trip; messages [#341](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com), [#271](https://www.postgresql.org/message-id/CAD21AoBuEqcz2_%2BdpA3WTiDUF%3DFgudPBKwM%2BnvH%2BqHT-k4p5mA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 4. Wire protocol format code for custom formats is not defined

A client driver runs COPY TO STDOUT with an Arrow format and reads CopyOutResponse. If the server reports the overall format as text (0), a driver that treats text COPY data as character data in client_encoding can mangle the binary stream. The user finds out when the received file does not parse.

CopyOutResponse and CopyInResponse carry an overall format and per-column codes that only distinguish text and binary. Message #8 asked how column-oriented formats would work at the protocol level, and the visible material does not answer it. Message #341 tested \copy only with ndjson, which is text, so a binary format through the frontend path is untested.

**Question:** Which overall and per-column format codes does the server send in CopyOutResponse and CopyInResponse for a custom format, and who chooses them?

<sub>fixable, level 2, wire protocol; messages [#8](https://www.postgresql.org/message-id/ee94f987-f4ca-4a0c-96d7-909465d20808%40manitou-mail.org), [#341](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com); `replication-ondisk-3`</sub>

### 5. Exported COPY I/O functions may block future COPY speedups

A developer wants to rework COPY FROM buffering, for example by parsing directly from raw_buf in batches. Extensions now call CopyGetData and read copy_file directly, as message #341 proposes for Parquet. The rework either breaks those extensions or has to keep the old per-call interface alive.

Message #309 asks about the COPY speedups this design forecloses, separately from the measured overhead. Message #310 answers only about indirection layers, and message #332 says that question is still open. Exporting CopyGetData, CopySendData and copy_file access fixes the current I/O layer as a contract.

**Question:** Which planned changes to COPY buffering or I/O would still be possible once CopyGetData, CopySendData and copy_file are part of the extension API?

<sub>fixable, level 2, performance on the hot path; messages [#309](https://www.postgresql.org/message-id/awcj7u3i5nf5yksmcoq24mtodd7kbmxljaygf72txhu6jxcd34%40ckdj2omq7v6u), [#310](https://www.postgresql.org/message-id/CAD21AoAQkjU%3Do0nX4y0jtX0BnsrqA04g2ABqrUwjT88YeEWarA%40mail.gmail.com), [#332](https://www.postgresql.org/message-id/c36d218a-bb38-42b9-9076-cb75b8984a39%40vondra.me), [#340](https://www.postgresql.org/message-id/179011382359.2771650.8713483121447030047%40gmail.com), [#341](https://www.postgresql.org/message-id/20261110121106.55499-1-joaovictor.rito%40gmail.com); `maintenance-fit-4`</sub>

## Questions for the author

1. What rule makes core reject a COPY option, including ON_ERROR, DELIMITER and future options, that a custom format does not declare support for?
2. Will the series include an in-tree format that reads and writes real rows through the exported I/O functions on file, frontend and PROGRAM paths?
3. How does a file_fdw table that names a custom format survive pg_dump, pg_restore and pg_upgrade when the library is not preloaded?
4. Which overall and per-column format codes does the server send in CopyOutResponse and CopyInResponse for a custom format, and who chooses them?
5. May a COPY FROM format read copy_file directly, and which changes to COPY buffering stay possible once CopyGetData and CopySendData are exported?

## Touch points

- **on-disk format / pg_upgrade:** file_fdw stores a custom format name in its options, and pg_upgrade's schema restore must resolve it.
- **wire protocol:** COPY TO STDOUT and FROM STDIN with a custom format send CopyOutResponse and CopyInResponse format codes.
- **catalog changes:** see objection replication-ondisk-1: Custom format names stored in file_fdw options have no catalog anchor
- **ABI / extension API:** RegisterCopyCustomFormat(), copy_state.h with both COPY state structs, and proposed exports of the COPY I/O functions.
- **SQL-standard conflict or non-standard syntax:** The COPY option FORMAT accepts names registered by a loaded extension.
- **pg_dump / restore round-trip:** pg_dump emits file_fdw options that name a custom format, which restores only when the library is loaded.
- **psql UX conventions:** \copy passes a custom format through the frontend path; message #341 tested it only with a text format.
- **security / privileges:** Format code loaded through _PG_init() receives the relation and the full values arrays during COPY.
- **backpatchability:** Fields of the exported COPY state structs constrain what minor-release fixes may change.
- **maintenance burden ("who maintains this in 10 years"):** Every future COPY option and every change to COPY buffering must consider out-of-tree formats.
- **performance on the hot path:** Built-in text, csv and binary COPY run through the callback routines; benchmarks showed differences mostly within 5%.

## Who would commit it, and when

- Owner: qualified committer engaged. Sawada writes the current v3 series and Paquier supported the _PG_init() design. The owner needs to know the COPY code and extension API practice.
- Priority: fix design first; audience a sizeable minority; review cost high. Demand for JSON lines, Arrow and Parquet has lasted since 2018 and a committer writes the series. The open points are the option rule, a real in-tree format, the I/O contract and the protocol format code, across a 341-message thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 7 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
