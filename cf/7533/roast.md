# Design pre-review: libpq: Don't read past the end of a message when tracing it

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7533](https://commitfest.postgresql.org/patch/7533/); [thread](https://www.postgresql.org/message-id/flat/CAJnxHsiCrtervsGsxuEL2cynuRLe9JQyn9poiEH%3Dja_2GaEvAA%40mail.gmail.com), 3 of 3 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible ASan result, the fix puts the bound in the shared readers of fe-trace.c, and the cost falls only on clients that call PQtrace(). Confidence is medium because no committer has replied and the patch code was not read.

Not checked: The code was not read, so the claims that every reader stops at the message end and that well-formed traces are unchanged rest on the author's test results. The build pattern of 0003 and the exact trace output for truncated messages need a code-level review.

## Prior attempts

None found before 2026-10-23.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Which branches should get 0001 and 0002: master only, or back to v14 for 0001 and v18 for 0002?
2. If the series is back-patched, is the new note "mismatched message length: contents exceed length N" acceptable in PQtrace() output on stable branches?
3. Does 0002 drop the pre-3.0 startup error from the trace entirely, or could the tracer print it in some bounded form?
4. Is 007_trace.pl useful on builds without AddressSanitizer, or does it only check the printed output there?
5. Should the series wait for commitfest 7081, or go first and have 7081 rebase onto it?

## Touch points

- **wire protocol:** The series changes how libpq traces received protocol messages, and 0002 skips tracing a pre-3.0 startup error; the protocol itself does not change.
- **security / privileges:** The series closes client-side reads past a message that a hostile server or network can cause; security@ judged it not a vulnerability.
- **backpatchability:** 0001's bug dates from v14 and 0002's from v18, and the thread has not decided which branches get the fix.
- **maintenance burden ("who maintains this in 10 years"):** 0003 adds the test program libpq_trace_regress, which includes fe-trace.c, and the TAP test t/007_trace.pl.

## Who would commit it, and when

- Owner: silent. A committer who knows libpq's protocol code and fe-trace.c would have to own it. Message #2 Cc's Álvaro, but no committer has replied in the visible thread.
- Priority: proceed; audience niche; review cost medium. Only clients that call PQtrace() are affected, but this is a memory-safety fix with an ASan reproduction and an independent confirmation in message #3. The main patch is about 35 KB in one file, fe-trace.c, and the series adds no new user-facing surface beyond one trace note.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
