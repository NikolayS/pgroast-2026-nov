# Design pre-review: Stack-based tracking of per-node WAL/buffer usage

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6023](https://commitfest.postgresql.org/patch/6023/); [thread](https://www.postgresql.org/message-id/flat/CAP53PkzdBK8VJ1fS4AZ481LgMN8f9mJiC39ZRHqkFUSYq6KWmg%40mail.gmail.com), 49 of 49 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1a or 1b: the problem is stated, measured, and wanted by several reviewers. The surviving objections are fixable design points at level 2. replication-ondisk-1 asks for a backstop when an extension calls InstrStart without abort handling, and maintenance-fit-1 asks for a committed test that catches lost abort-time totals. The abort model needs sign-off from a committer who knows the executor and resource owners.

Not checked: The code was not read, so the abort handling, the resource owner release order and the parallel worker accumulation could not be checked. The README hunk was truncated, and only 3 of 49 messages were shown, so later answers in the thread may already cover some questions.

## Prior attempts

- [Reduce timing overhead of EXPLAIN ANALYZE using rdtsc](https://www.postgresql.org/message-id/flat/20200612232810.f46nbqkdhbutzqdg@alap3.anarazel.de) (similar idea; Returned with feedback, 2024-02-01). "Since the author or no one else showed interest in taking it forward and the patch had no activity for more than 1 year, I have changed the status to RWF." ([message](https://www.postgresql.org/message-id/CALDaNm2awLBioTdQH%3Daj8Yp7%2BMwR%3DvzpzHrKtgjAnUN1esLmvg%40mail.gmail.com)) Different now: P1 cut the timing cost of EXPLAIN ANALYZE by reading the TSC. This proposal cuts the buffer and WAL counting cost by replacing the BufferUsageAccumDiff diff with a stack. P1 closed from inactivity, and this thread is active at v18.
- [Sampling-based timing for EXPLAIN ANALYZE](https://www.postgresql.org/message-id/flat/CAP53PkxXMk0j-+0=YwRti2pFR5UB2Gu4v2Oyk8hhZS0DRART6g@mail.gmail.com) (similar idea; Returned with feedback, 2023-07-03). reason not recorded Different now: P22 tried to lower EXPLAIN ANALYZE overhead by sampling timings. This proposal keeps exact per-node counts and changes how buffer and WAL usage reach each node. No reason is recorded for P22.

## Objections a reviewer is likely to raise

### 1. Extension callers of the plain Instrumentation API can leave a dangling stack

An extension that read pgBufferUsage switches to InstrStart/InstrStop as the 0005 commit message advises, but adds no PG_TRY/PG_FINALLY. An error between the two calls leaves instr_stack.current pointing at memory freed by the abort. Later buffer and WAL counter updates in the same backend write into that freed memory, and nothing reports it.

The 0005 commit message says plain Instrumentation "requires appropriate abort handling in all cases", and the 0004 message limits abort safety to QueryInstrumentation or explicit PG_TRY/PG_FINALLY. Message #26 confirms that without this reset the stack is corrupted after an abort. The material shows no backstop, such as a reset to instr_top or an assertion in AbortTransaction, for callers that get this wrong. Today an extension that diffs pgBufferUsage cannot corrupt memory this way, so the API change turns a counting mistake into a safety bug.

**Question:** What resets or checks instr_stack at transaction abort or error recovery when an extension calls InstrStart without PG_FINALLY or a resource owner?

<sub>fixable, level 2, ABI / extension API; messages [#26](https://www.postgresql.org/message-id/CAP53Pkx%2BHZ2OSGDHmNgSDisXdjGp4KFygaX%2BGaQcFEuXHRXg%3Dg%40mail.gmail.com), [#49](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com); `replication-ondisk-1`</sub>

### 2. Lost abort-time totals are caught only by a test module not meant for commit

A later patch adds a new kind of instrumentation, for example per-trigger or per-receiver, and does not register it as a child of the QueryInstrumentation. When that code errors out, its buffer activity is not added to the parent. Regression tests pass, and session or pg_stat_statements totals undercount without any visible error.

The trigger case in messages #21 and #23 and the EXPLAIN (SERIALIZE) case in message #15 were both missing from abort accounting. Both were found only through test_session_buffer_usage, which the author calls test-only and not meant for commit (messages #11, #19, #32). Message #26 says a stack that is not reset does fail regression tests, but lost accumulation does not. Once pgBufferUsage is retired in 0005, these abort totals are the only session-level number, so a silent loss is harder to spot than it is today.

**Question:** Which committed test fails when a new instrumentation user skips registering with its QueryInstrumentation and then errors out?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#11](https://www.postgresql.org/message-id/CAP53PkzGbyeJMLDAcvMRgzXPXYsYXZr3SBg0UwhfkYjqu8oK_g%40mail.gmail.com), [#15](https://www.postgresql.org/message-id/CAP53PkxAS-rpp0kcOjg--q5NVNnRC%2B0kh1%2BgB8gnRNqacTqF5A%40mail.gmail.com), [#21](https://www.postgresql.org/message-id/CAN4CZFMk10YCixVTnt%2BuBaUH_BH1R%3DEr-PnPs2YDBhFjuE3K-Q%40mail.gmail.com), [#23](https://www.postgresql.org/message-id/CAP53PkyEPa%3DaFZrd0crLgRJm1KGS0ykJ%2B1i2CwDfDNFKpCc%2BXQ%40mail.gmail.com), [#26](https://www.postgresql.org/message-id/CAP53Pkx%2BHZ2OSGDHmNgSDisXdjGp4KFygaX%2BGaQcFEuXHRXg%3Dg%40mail.gmail.com), [#32](https://www.postgresql.org/message-id/CAP53Pky2%3D31B1AS9vg%3DCa9308_hb0H4g968cSKxiFgT0moJfYg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. Two counting models remain, buffers on the stack and WAL on a global

A developer adds a new counter, or moves pgstat WAL accounting, and has to decide between the stack and pgWalUsage. INSTR_WALUSAGE_ADD writes both. If a fix updates only one of the two paths, pg_stat_wal and EXPLAIN WAL numbers drift apart, and nobody notices until a user compares them.

The 0005 commit message keeps pgWalUsage because pgstat and heap_page_prune_and_freeze depend on it, and message #3 calls removing it "interesting". The material gives no plan or release for retiring it. Every WAL record now updates two places, and the thread shows no measurements with WAL ON. Message #49 also cites an existing pg_stat_wal bug around these globals.

**Question:** What is the plan and target release for retiring pgWalUsage, and what does WAL ON overhead measure today?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#3](https://www.postgresql.org/message-id/CAP53Pkw1RpyAVsvJwUOMgGnVoQUibMk%2BEB%2Bw-8WqY8kzsC4WjA%40mail.gmail.com), [#49](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com); `maintenance-fit-2`</sub>

### 4. "Table Buffers" label adds a third name next to "Heap" fields

Release 20 ships "Table Buffers" next to "Heap Fetches" and "Heap Blocks". Plan visualizers and log parsers then match on the text label and the JSON group key. A later rename to fix the inconsistency breaks those tools, so the project keeps the mixed naming for good.

In message #11 the author asks whether "Table Buffers" fits with "Heap Fetches" and "Heap Blocks", and message #12 says this is still open. v18 keeps the label, and structured output gets the new "Table Buffers" and "Table I/O Timings" groups (message #17). EXPLAIN labels and JSON keys are a contract that external tools read, so the naming should be settled before commit.

**Question:** Which naming will EXPLAIN use for table access across nodes, and will "Heap Fetches" and "Heap Blocks" be renamed or kept?

<sub>fixable, level 2, SQL-standard conflict or non-standard syntax; messages [#11](https://www.postgresql.org/message-id/CAP53PkzGbyeJMLDAcvMRgzXPXYsYXZr3SBg0UwhfkYjqu8oK_g%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAP53PkxrmpECzVFpeeEEHDGe6u625s%2BYkmVv5-gw3L_NDSfbiA%40mail.gmail.com), [#17](https://www.postgresql.org/message-id/CAP53PkxFP7i7-wy98ZmEJ11edYq-RrPvJoa4kzGhBBjERA4Nyw%40mail.gmail.com), [#49](https://www.postgresql.org/message-id/CAP53Pkwd6Gd4OL6cgh449wtdznWitz_V%3D-PRoswLHOzsMbcftA%40mail.gmail.com); `standards-compat-2`</sub>

## Questions for the author

1. What resets or checks instr_stack at transaction abort when an extension calls InstrStart without PG_TRY/PG_FINALLY or a resource owner?
2. Which committed test fails when a new instrumentation user does not register with its QueryInstrumentation and then errors out?
3. Is test_session_buffer_usage meant for commit, and if not, what replaces its coverage of top-level totals after aborts?
4. What is the plan and target release for retiring pgWalUsage, and what overhead does the stack patch show with WAL ON?
5. Which naming will EXPLAIN use for table access across nodes, and will "Heap Fetches" and "Heap Blocks" be renamed or kept?

## Touch points

- **WAL / replication / standby:** 0002 changes how xlog.c updates WAL usage counters, and 0004 touches replication/logical/worker.c; no WAL record changes.
- **ABI / extension API:** 0005 removes pgBufferUsage and BufferUsageAccumDiff and adds QueryInstrumentation, NodeInstrumentation and the INSTR_* macros.
- **SQL-standard conflict or non-standard syntax:** see objection standards-compat-2: "Table Buffers" label adds a third name next to "Heap" fields
- **psql UX conventions:** 0007 adds a "Table Buffers" line and structured "Table Buffers" and "Table I/O Timings" groups to EXPLAIN ANALYZE output.
- **backpatchability:** The series rewrites instrument.c and the counter updates in bufmgr.c, so later back-patched fixes in those areas will not apply cleanly.
- **maintenance burden ("who maintains this in 10 years"):** Every new instrumentation user must handle aborts through a resource owner or PG_TRY/PG_FINALLY, and pgWalUsage stays as a second counting model.
- **performance on the hot path:** Buffer and WAL counter updates in bufmgr.c, localbuf.c and xlog.c now write through the current instrumentation stack entry.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund pushed the preparatory patches and reviewed the stack design, and Heikki Linnakangas reviewed an earlier version. The stack patch still needs a committer who knows executor instrumentation, resource owners and abort handling; v18 has no reply yet.
- Priority: proceed; audience a sizeable minority; review cost high. EXPLAIN (ANALYZE, BUFFERS) users and tools like pg_stat_statements gain from the lower overhead, and two reviewers want the table buffer split. The series is eight patches across the executor, buffer manager, WAL insertion and several access methods, and the abort model adds rules that future instrumentation users must follow.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 4.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
