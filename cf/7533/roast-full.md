# Design pre-review: libpq: Don't read past the end of a message when tracing it

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7533](https://commitfest.postgresql.org/patch/7533/) (Needs review); authors: Ana Bobillo (anabobillo3)
- Thread: [libpq: Don't read past the end of a message when tracing it](https://www.postgresql.org/message-id/flat/CAJnxHsiCrtervsGsxuEL2cynuRLe9JQyn9poiEH%3Dja_2GaEvAA%40mail.gmail.com), started 2026-10-23
- Messages read: 3 of 3
- Patch prose read: v1-0001-libpq-Don-t-read-past-the-end-of-a-message-when-t.patch, v1-0002-libpq-Don-t-trace-a-pre-3.0-error-message-as-a-pr.patch, v1-0003-libpq-Add-a-test-of-tracing-malformed-protocol-me.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated with a reproducible ASan result, the fix puts the bound in the shared readers of fe-trace.c, and the cost falls only on clients that call PQtrace(). Confidence is medium because no committer has replied and the patch code was not read.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read, so the claims that every reader stops at the message end and that well-formed traces are unchanged rest on the author's test results. The build pattern of 0003 and the exact trace output for truncated messages need a code-level review.

## The proposal, restated

**Problem.** The protocol tracer in fe-trace.c trusts counts, lengths and string terminators inside server messages, so a malformed message makes a client using PQtrace() read past the message in its input buffer. Messages libpq discards without parsing are read by the tracer alone, so the checks in fe-protocol3.c do not cover them. Security@ judged it not a vulnerability because it needs a hostile server or network.

**Proposed design.** 0001 passes the end of the message to every reader in fe-trace.c, which stop there and report "mismatched message length: contents exceed length N". It also treats any negative DataRow length as NULL and fixes how CopyBothResponse reads its format count. 0002 stops tracing a pre-3.0 error message during startup, and 0003 adds an AddressSanitizer-oriented TAP test.

**User-visible surface.**
- PQtrace() output: malformed messages are printed as far as they go, with the new note "mismatched message length: contents exceed length N".
- PQtrace() output: a DataRow field with any negative length is printed as NULL.
- PQtrace() output: a pre-3.0 error message received at connection startup is no longer traced.
- New test program libpq_trace_regress and TAP test t/007_trace.pl.

**What the author claims.**
- Well-formed messages are traced as before; libpq and libpq_pipeline suites pass with unchanged traces.
- On master with ASan, 007_trace.pl fails 33 checks with heap-buffer-overflow reports; with the series all 84 pass.
- 0001's bug dates from v14; 0002's from b8b3f861fbd in v18.
- Commitfest 7081 (PQTRACE environment variable) would make these reads easier to reach.
- CFBot is green on all platforms.

**Thread state.** The author pinged after two weeks with no reply. Zsolt Parragi confirmed the series fixes his own fuzzing cases and promised a full review; no committer has replied and back-patching is not decided.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-23; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | The series changes how libpq traces received protocol messages, and 0002 skips tracing a pre-3.0 startup error; the protocol itself does not change. |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The series closes client-side reads past a message that a hostile server or network can cause; security@ judged it not a vulnerability. |
| backpatchability | **yes** | 0001's bug dates from v14 and 0002's from v18, and the thread has not decided which branches get the fix. |
| maintenance burden ("who maintains this in 10 years") | **yes** | 0003 adds the test program libpq_trace_regress, which includes fe-trace.c, and the TAP test t/007_trace.pl. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which branches should get 0001 and 0002: master only, or back to v14 for 0001 and v18 for 0002?
2. If the series is back-patched, is the new note "mismatched message length: contents exceed length N" acceptable in PQtrace() output on stable branches?
3. Does 0002 drop the pre-3.0 startup error from the trace entirely, or could the tracer print it in some bounded form?
4. Is 007_trace.pl useful on builds without AddressSanitizer, or does it only check the printed output there?
5. Should the series wait for commitfest 7081, or go first and have 7081 rebase onto it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows libpq's protocol code and fe-trace.c would have to own it. Message #2 Cc's Álvaro, but no committer has replied in the visible thread.
- **Liability:** The owner signs for changed PQtrace() output on malformed messages, the back-patch decision, and a new test program that includes fe-trace.c and must build on all platforms. The liability is small and limited to the tracing code.
- **Author follow-up:** Good signs: the author pinged in message #2, offers to rework or split the series, and promises to rebase onto commitfest 7081 if it lands first.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Only clients that call PQtrace() are affected, but this is a memory-safety fix with an ASan reproduction and an independent confirmation in message #3. The main patch is about 35 KB in one file, fe-trace.c, and the series adds no new user-facing surface beyond one trace note.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found: the precedent search returned no earlier attempt at the same or a similar idea. The thread mentions only the security@ exchange (message #1), which sent the fix to the normal channels and reports no rejection. It also cites commitfest 7081, a related change to fe-trace.c that the author plans to rebase onto.
- standards-compat: 0 objection(s). Nothing in the user-facing contract breaks: well-formed messages trace as before, and only malformed messages or a pre-3.0 startup error, mainly from out-of-support servers, trace differently. The proposal adds no syntax, GUC, catalog object or pg_dump state. A committer may still ask whether the new trace note is acceptable on back branches.
- replication-ondisk: 0 objection(s). The series has no persistent or cross-node state: no on-disk format, WAL, catalog or replication change. It does not change the wire protocol either; it only changes how libpq prints received messages in PQtrace() output, and the exported PQtrace() API stays the same.
- security: 0 objection(s). The series closes client-side over-reads of server-controlled data and adds no new privilege, default or exposure. It keeps the trace output within each message, so it reduces what a hostile server can make the client read. Back-patch scope is open, but I found no security failure scenario against the design.
- maintenance-fit: 0 objection(s). From the maintenance angle this is a contained bugfix and fits core. 0001 puts the bound inside the shared readers (Byte1, Int16, Int32, String, Nchar), so a printer added later for a new message type cannot read past the message even if its author forgets a check. The fix adds no cost for clients that do not trace, adds no GUC, catalog change or dependency, and changes no API. The test program includes fe-trace.c the same way test-oauth-curl.c already includes oauth-curl.c. Changes to the trace output affect only malformed messages, and PQtrace() output is a debugging aid, so tools that parse it are not a reason to keep reading past the buffer. Two points are open but are not objections: which branches get the fix, and the rebase against CF 7081. No committer has replied yet, and Zsolt Parragi's review is still pending.
- already-exists: 0 objection(s). This is a bug fix in fe-trace.c, and no existing feature or extension stops PQtrace() from reading past a malformed message. The checks in fe-protocol3.c do not cover messages that libpq discards without parsing, and no mechanism is duplicated. The test copies the existing test-oauth-curl.c pattern.
- demand: 0 objection(s). This is a memory-safety bugfix in PQtrace(), not a feature, so the demand test does not apply. Any client that calls PQtrace() is exposed, and Zsolt Parragi found the same reads by fuzzing on his own (message #3).

### Precedents retrieved

- P1: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P2: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant; reason summarised from its thread
- P3: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P4: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P5: [Transaction traceability - txid_convert_if_recent(bigint)](https://commitfest.postgresql.org/patch/748/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P6: [Revert removal of trace_userlocks, because userlocks aren't gone.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71b2b657c034743f81ae7906d7b0fcc8e9a0a6a3) (revert, Reverted, 2011-11-10) - judged not relevant
- P7: [Add new protocol message to change GUCs to be able to change protocol extension parameters](https://commitfest.postgresql.org/patch/4736/) (cf, Withdrawn, 2025-02-22) - judged not relevant; reason summarised from its thread
- P8: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant; reason summarised from its thread
- P9: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P10: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - judged not relevant; reason summarised from its thread
- P11: [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (cf, Withdrawn, 2024-05-14) - judged not relevant; reason summarised from its thread
- P12: [Use "protocol options" name instead of "protocol extensions" everywhere](https://commitfest.postgresql.org/patch/5341/) (cf, Rejected, 2025-03-11) - judged not relevant
- P13: [Password identifiers, protocol aging and SCRAM protocol](https://commitfest.postgresql.org/patch/535/) (cf, Returned with feedback, 2016-03-22) - judged not relevant
- P14: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P15: [meson/testwrap: support EXTRA_REGRESS_OPTS environment variable](https://commitfest.postgresql.org/patch/6711/) (cf, Withdrawn, 2026-05-04) - judged not relevant
- P16: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1317/) (cf, Returned with feedback, 2018-11-29) - judged not relevant
- P17: [reduce impact of lengthy startup and checkpoint tasks](https://commitfest.postgresql.org/patch/3448/) (cf, Returned with feedback, 2023-09-01) - judged not relevant
- P18: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - judged not relevant
- P19: [Revert "Test replay of regression tests."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dccee0f2b72648b8b3c42883a7d99210eb813f30) (revert, Reverted, 2022-01-14) - judged not relevant
- P20: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P21: [Revert no-op changes to BufferGetPage()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a343e223a5c33a7283a6d8b255c9dbc48dbc5061) (revert, Reverted, 2016-04-20) - judged not relevant
- P22: [Make installcheck-world in a clean environment](https://commitfest.postgresql.org/patch/1672/) (cf, Withdrawn, 2019-07-30) - judged not relevant
- P23: [Permute underscore separated components of columns before fuzzy matching](https://commitfest.postgresql.org/patch/4282/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P24: [Detect test suite sideeffects in pg_regress](https://commitfest.postgresql.org/patch/4321/) (cf, Withdrawn, 2024-04-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 10; wall time: 59 s; cost reported by the CLI: $0.61
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 47 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:06:23.717Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
