# Design pre-review: Add an ETW backend for the static probes on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7566](https://commitfest.postgresql.org/patch/7566/) (Needs review); authors: Bryan Green (lispyone)
- Thread: [\[PATCH\] Add an ETW backend for the static probes on Windows](https://www.postgresql.org/message-id/flat/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com), started 2026-10-30
- Messages read: 1 of 1
- Patch prose read: v1-0001-Add-an-ETW-backend-for-the-static-probes-on-Windo.patch, v1-0002-ci-Enable-the-ETW-build-in-the-Windows-Visual-Stu.patch, v1-0003-Document-the-ETW-probe-backend.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the goal follows existing practice: the same probes already have DTrace and SystemTap backends. The remaining points are fixable design questions. They are the default-on exposure of query text (security-1), the UTF-8 tag on strings sent in the server encoding (standards-compat-2), the second probe file (maintenance-fit-1) and the unstated schema stability rule (standards-compat-1). Demand evidence is thin (demand-1), but parity is a fair reason to keep reading.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, three things could not be checked: how EXEC_BACKEND children register the provider, whether Windows extensions that use the probe macros can link to it, and the overhead with VERBOSE lock and buffer events enabled.

## The proposal, restated

**Problem.** Windows builds use the dummy probes.h, so the static probes are compiled out and cannot be traced there. DTrace and SystemTap cover other platforms, but Postgres has no backend for Event Tracing for Windows (ETW).

**Proposed design.** The probe macros expand to TraceLogging events from one provider, "PostgreSQL.Server", registered once per process and removed at process exit. A new data file, probes_etw.txt, gives each probe a keyword, a level and argument names. probes.d stays the source of truth, and the build fails on every platform if the two files disagree.

**User-visible surface.**
- New meson option -Detw (auto, enabled, disabled), default auto, mutually exclusive with -Ddtrace; MinGW builds keep the dummy probes.
- ETW provider "PostgreSQL.Server" with GUID ea7811de-e757-5488-7ad6-2b057cced37d, documented as a stable contract that never changes.
- Event names match the DTrace probe names with underscores, and fields are named, with keywords such as QUERY, LOCK and BUFFER and levels INFO or VERBOSE.
- New file src/backend/utils/probes_etw.txt, which every new probe needs, plus a build check on all platforms.
- String arguments are tagged UTF-8 but passed in the server encoding, and are clipped to 32kB.

**What the author claims.**
- No new probes and no change to probes.d or the call sites; only the macro expansion changes.
- On pgbench -S, an idle ETW build was within 0.5% of the dummy build, which the author calls noise.
- Probe arguments are evaluated only when a session has enabled the event.
- Events show up in wpr and WPA in the same trace as kernel disk and scheduler events.

**Thread state.** Only the first message is visible, and no one has replied. The CommitFest entry is in Needs review for version 20.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-30; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Default-on ETW exposes all sessions' query text to Performance Log Users

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A packager ships an MSVC build, where -Detw=auto turns the provider on. An operator who is in Performance Log Users, but not an administrator and not the service account, runs the logman command from message #1. The .etl file then holds every backend's query text, including literals and `alter role ... password` statements, and no one in Postgres can see that this happened.

On other platforms the probes are off unless the build uses --enable-dtrace, and tracing them needs root or similar rights. This patch turns the probes on by default in MSVC builds. Message #1 says capture needs only Performance Log Users membership, a group meant for delegated monitoring. The query, query__parse and statement__status probes carry text that log_statement keeps under superuser control. The docs hunks give no security note and do not say who may enable the provider.

Thread: [#1 (Bryan Green, 2026-10-30)](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com)

**Question:** Why is it acceptable for Performance Log Users members to read every session's query text by default, and should the QUERY strings or the default be gated?

Counter-check: held. Message #1 says membership in Performance Log Users is enough to capture, and the QUERY events carry full query text. The visible docs give no security note, and the default is auto.

### 2. Strings tagged as UTF-8 but sent in the server encoding

fixable · level 2, rubric item 12 · design, security / privileges · `standards-compat-2`

A DBA traces a cluster whose databases use WIN1252 or EUC_JP and opens the trace in WPA or tracerpt. These tools decode query text as UTF-8, so non-ASCII characters show as replacement characters or the field fails to decode. A query clipped at 32kB can also end in a partial multibyte character.

The commit message says string arguments are tagged UTF-8 but passed in the server encoding, so the tag is wrong for every non-UTF-8 database. Windows clusters often use legacy code pages, so this affects a real share of the users this backend targets. TraceLogging has field types for ANSI strings and counted binary data that would not make a false encoding claim. The proposal does not say how a consumer should decode these fields.

Thread: [#1 (Bryan Green, 2026-10-30)](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com)

**Question:** How does a WPA or tracerpt user get correct query text from a WIN1252 or EUC database, and how is a cut multibyte character handled?

Counter-check: held. The commit message itself says strings are tagged UTF-8 but passed in the server encoding and clipped at 32kB. The material does not show how common legacy encodings are on Windows, but the wrong tag stands for any non-UTF-8 database.

### 3. Every probe change on every platform must now edit probes_etw.txt

fixable · level 2, rubric item 4 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A Linux developer adds an argument to an lwlock probe in probes.d. The build fails on their machine until they also edit probes_etw.txt and choose an ETW keyword, level and field name for a facility they never use. Argument descriptions now live in probes.d, monitoring.sgml and probes_etw.txt.

The commit message makes the consistency check run in meson and make builds without ETW, so the cost falls on all developers while only MSVC Windows users gain (item 4). The check fails loudly, which answers item 9, but it adds a third copy of each argument's type, plus a field name that the file header says is copied by hand from monitoring.sgml. A committer would likely ask whether keyword, level and names could be derived or defaulted, so that probes.d stays the only file to edit.

Thread: [#1 (Bryan Green, 2026-10-30)](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com)

**Question:** Why can probes_etw.txt not be generated, or given defaults, from probes.d and the existing documentation, so a non-Windows probe change needs no second file?

Counter-check: held. The commit message confirms the check runs in builds without ETW, and the file header says field names are copied from monitoring.sgml. The proposal does not say why keyword, level and names cannot be defaulted.

### 4. Keyword bits and event fields become a contract with no stated stability rule

fixable · level 2, rubric item 6 · design, maintenance burden ("who maintains this in 10 years"), backpatchability · `standards-compat-1`

An admin saves a logman or wpr profile that filters on keyword mask 0x3 and level 4, as message #1 shows. A later release reorders the PG_KW_* bits or renames a probe argument in probes_etw.txt. The saved profile then captures the wrong events or fields, and the admin finds out only when a trace comes back empty or mislabeled.

The docs promise that the provider name and GUID never change. Capture profiles also depend on the keyword bit values, the levels, the event names and the field names, and the example in message #1 hard-codes the mask 0x3. The proposal does not say whether these stay stable across major versions or may change with probes.d. Without a stated rule, users will treat them as stable, and every later probe change becomes a compatibility question.

Thread: [#1 (Bryan Green, 2026-10-30)](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com)

**Question:** Which parts of the event schema (keyword bits, levels, event and field names) are promised stable across major versions, and how are users told when they change?

Counter-check: held. The visible docs promise stability only for the provider name and GUID. The example in message #1 hard-codes the keyword mask 0x3, so saved profiles depend on bits and names that have no stated rule.

### 5. No evidence that Windows users want to trace the static probes

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

The project adds and documents a provider GUID as a contract that "never changes", plus a second probe file that has to be kept up to date. If few Windows users ever start an ETW session for it, the project keeps that contract and file for years for a feature with little use.

Message #1 gives parity with other platforms as the only reason. It cites no field report, no -general or -bugs thread, no second user and no production story. Even on Linux and macOS the static probes need a build option that is off by default, so their user base may be small already. No one has replied in the thread.

Thread: [#1 (Bryan Green, 2026-10-30)](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com)

**Question:** Which Windows users or support cases needed these probes, what did they use instead, and will the common Windows installers ship with -Detw enabled?

Counter-check: held. Message #1 gives platform parity as the only reason and names no user or support case. Parity explains the goal, but it does not show that enough people will use it to justify a permanent GUID contract and a second probe file.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The probe macros in probes.h now expand to TraceLogging calls on a provider handle, which Windows extensions that use the macros would need. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | QUERY events carry query text, and message #1 says membership in Performance Log Users is enough to capture them. |
| backpatchability | **yes** | see objection standards-compat-1: Keyword bits and event fields become a contract with no stated stability rule |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds probes_etw.txt, a build check on every platform, a permanent provider GUID and a dependency on a Windows SDK header. |
| performance on the hot path | **yes** | Probe sites in lwlock and buffer code become live enabled-checks in default MSVC builds. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No evidence that Windows users want to trace the static probes |

## 5. Questions the author must answer before anyone reviews code

1. Who may enable the provider, and should the query text fields or the auto default be gated, given that Performance Log Users members can capture every session's queries?
2. How does a WPA or tracerpt user read query text from a WIN1252 or EUC_JP database, and what happens to a multibyte character cut at 32kB?
3. Can the keyword, level and field names be generated or defaulted from probes.d, so that a probe change on a non-Windows platform needs no edit to probes_etw.txt?
4. Which parts of the event schema (keyword bits, levels, event names, field names) stay stable across major versions, besides the provider name and GUID?
5. Which Windows users or support cases needed these probes, and what did they use instead?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the Windows MSVC build and process startup under EXEC_BACKEND, and who can test with wpr or logman.
- **Liability:** The owner signs for a provider name and GUID documented as never changing, an event schema users will save in capture profiles, a dependency on a Windows SDK header, and a build check that every later probe change must pass.
- **Author follow-up:** No sign either way; only message #1 exists. The patch set includes a CI change and docs.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The benefit goes to Windows MSVC users who trace, and the main patch is about 32 KiB across the build system, postmaster.c and miscinit.c. The security default and the string encoding should be settled before code review, since both change what the provider emits.

## Appendix

### Refuted in the counter-check

- `standards-compat-3` Default auto differs from dtrace, which defaults to disabled. Refuted (not specific): Duplicate of security-1, which states the same default-on consequence with a concrete scenario. The consistency part is weak, because the gssapi option in the same meson_options.txt hunk also defaults to auto.
- `maintenance-fit-2` Default auto differs from dtrace, which defaults to disabled. Refuted (no failure scenario): The objection names no workload that would regress, and pgbench -S already passes through the lwlock and buffer probe sites with the provider idle. The cost with a session active falls only on users who trace. The default question is covered by security-1.
- `maintenance-fit-3` Windows-only backend with no engaged committer to maintain it. Refuted (not specific): A thread with no reply after two weeks fits any new Windows patch, and the rubric puts ownership under committability, separate from the design verdict. The SDK breakage scenario is speculative, since the proposal does not show that TraceLogging macros change behaviour.
- `demand-2` Every probe author pays for probes_etw.txt, on every platform. Refuted (not specific): Duplicate of maintenance-fit-1, which makes the same point about probes_etw.txt in more detail.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. [P3] adds arguments to the LWLock tracepoints and stalled on a macOS build failure and missing docs, which does not apply to a Windows backend. [P6] and [P8] are DTrace reverts with no recorded reason, and message #1 cites no earlier discussion of ETW.
- standards-compat: 3 objection(s). Objections are listed above. The idea of an ETW backend fits existing practice, and none of these objections is fatal.
- replication-ondisk: 0 objection(s). The proposal has no persistent or cross-node state: no on-disk format, WAL, catalog, replication or wire protocol change, and probes.d and its call sites stay the same. Without the code I could not check two things: whether extensions built on Windows that call TRACE_POSTGRESQL_* macros can link to the provider handle, and how EXEC_BACKEND children register.
- security: 1 objection(s). One objection on exposure. Not evaluated without the code: whether an unprivileged OS user can slow the server by enabling the VERBOSE lock and buffer events, because the material gives no overhead figure for an active trace session.
- maintenance-fit: 3 objection(s). Not applicable; three fixable objections raised.
- already-exists: 0 objection(s). No existing feature gives this result on Windows: the probes compile to the dummy probes.h there, and no extension or hook can supply them, since the macro expansion happens at compile time. The patch extends the existing probes.d mechanism rather than adding a second one, and I am not sure enough about any external Windows tool that reads these static probes to name it.
- demand: 2 objection(s). Both objections point to missing demand evidence, not to a flaw in the idea. Measured overhead with a trace session active, and security, are left to other angles.

### Precedents retrieved

- P1: [Revert "Temporarily add some probes of tenk1's relallvisible in create_index.sql."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=91998539b227dfc6dd091714da7d106f2c95a321) (revert, Reverted, 2022-04-15) - judged not relevant
- P2: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant; reason summarised from its thread
- P3: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - used in the review; reason summarised from its thread
- P4: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P5: [Revert "Have dtrace depend on object files directly, not objfiles.txt"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=08fbad0afd62690cc82990c0504529ef238ac24d) (revert, Reverted, 2015-10-15) - judged not relevant
- P6: [Revert DTrace patch from Robert Lor](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0e550ff6174ebe8bab15b1218fa336e484513a61) (revert, Reverted, 2009-04-02) - used in the review
- P7: [Revert to the pre-8.2 method of probing for libm, that is, always include it if it links properly.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=02f37bd8c4bfdd35869007d142267cd9a1f671b9) (revert, Reverted, 2006-11-06) - judged not relevant
- P8: [Revert ill-considered restriction of dtrace support to Solaris only.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=db13a81ab44f5ead85a4e77b30006c81bb6d20af) (revert, Reverted, 2009-09-04) - used in the review
- P9: [Revert removal of trace_userlocks, because userlocks aren't gone.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71b2b657c034743f81ae7906d7b0fcc8e9a0a6a3) (revert, Reverted, 2011-11-10) - judged not relevant
- P10: [Windows meson build](https://commitfest.postgresql.org/patch/5522/) (cf, Returned with feedback, 2026-03-27) - judged not relevant; reason summarised from its thread
- P11: [Implement <null treatment> for window functions](https://commitfest.postgresql.org/patch/2628/) (cf, Returned with feedback, 2021-03-25) - judged not relevant; reason summarised from its thread
- P12: [Transaction traceability - txid_convert_if_recent(bigint)](https://commitfest.postgresql.org/patch/748/) (cf, Returned with feedback, 2016-09-29) - judged not relevant
- P13: [Fix severe performance regression with gettext 0.20+ on Windows](https://commitfest.postgresql.org/patch/6491/) (cf, Returned with feedback, 2026-03-27) - judged not relevant; reason summarised from its thread
- P14: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P15: [CREATE COLLATION default provider](https://commitfest.postgresql.org/patch/4366/) (cf, Withdrawn, 2024-01-21) - judged not relevant; reason summarised from its thread
- P16: [DOCS for generated column replication](https://commitfest.postgresql.org/patch/5502/) (cf, Rejected, 2025-03-12) - judged not relevant
- P17: [generated columns](https://commitfest.postgresql.org/patch/1271/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P18: [Provide the facility to set binary format output for specific OID's per session](https://commitfest.postgresql.org/patch/3777/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P19: [ICU as default collation provider](https://commitfest.postgresql.org/patch/1543/) (cf, Returned with feedback, 2019-02-04) - judged not relevant
- P20: [Generate Doxygen documentation](https://commitfest.postgresql.org/patch/4641/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P21: [Improvements to Meson docs](https://commitfest.postgresql.org/patch/4208/) (cf, Returned with feedback, 2024-01-20) - judged not relevant
- P22: [Add generate_series(date,date,\[step\])](https://commitfest.postgresql.org/patch/492/) (cf, Returned with feedback, 2016-03-17) - judged not relevant
- P23: [Meson build updates](https://commitfest.postgresql.org/patch/4410/) (cf, Withdrawn, 2023-09-01) - judged not relevant
- P24: [change SOFT_ERROR_OCCURRED to a static inline function](https://commitfest.postgresql.org/patch/6666/) (cf, Withdrawn, 2026-08-07) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 148 s; cost reported by the CLI: $1.84
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 7 from cache, 1 summarised now, 0 failed; quotes: 1 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 92 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:11:16.244Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
