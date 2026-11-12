# Design pre-review: Add an ETW backend for the static probes on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7566](https://commitfest.postgresql.org/patch/7566/); [thread](https://www.postgresql.org/message-id/flat/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com), 1 of 1 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the goal follows existing practice: the same probes already have DTrace and SystemTap backends. The remaining points are fixable design questions. They are the default-on exposure of query text (security-1), the UTF-8 tag on strings sent in the server encoding (standards-compat-2), the second probe file (maintenance-fit-1) and the unstated schema stability rule (standards-compat-1). Demand evidence is thin (demand-1), but parity is a fair reason to keep reading.

Not checked: Without the code, three things could not be checked: how EXEC_BACKEND children register the provider, whether Windows extensions that use the probe macros can link to it, and the overhead with VERBOSE lock and buffer events enabled.

## Prior attempts

None found before 2026-10-30.

## Objections a reviewer is likely to raise

### 1. Default-on ETW exposes all sessions' query text to Performance Log Users

A packager ships an MSVC build, where -Detw=auto turns the provider on. An operator who is in Performance Log Users, but not an administrator and not the service account, runs the logman command from message #1. The .etl file then holds every backend's query text, including literals and `alter role ... password` statements, and no one in Postgres can see that this happened.

On other platforms the probes are off unless the build uses --enable-dtrace, and tracing them needs root or similar rights. This patch turns the probes on by default in MSVC builds. Message #1 says capture needs only Performance Log Users membership, a group meant for delegated monitoring. The query, query__parse and statement__status probes carry text that log_statement keeps under superuser control. The docs hunks give no security note and do not say who may enable the provider.

**Question:** Why is it acceptable for Performance Log Users members to read every session's query text by default, and should the QUERY strings or the default be gated?

<sub>fixable, level 2, security / privileges; messages [#1](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com); `security-1`</sub>

### 2. Strings tagged as UTF-8 but sent in the server encoding

A DBA traces a cluster whose databases use WIN1252 or EUC_JP and opens the trace in WPA or tracerpt. These tools decode query text as UTF-8, so non-ASCII characters show as replacement characters or the field fails to decode. A query clipped at 32kB can also end in a partial multibyte character.

The commit message says string arguments are tagged UTF-8 but passed in the server encoding, so the tag is wrong for every non-UTF-8 database. Windows clusters often use legacy code pages, so this affects a real share of the users this backend targets. TraceLogging has field types for ANSI strings and counted binary data that would not make a false encoding claim. The proposal does not say how a consumer should decode these fields.

**Question:** How does a WPA or tracerpt user get correct query text from a WIN1252 or EUC database, and how is a cut multibyte character handled?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com); `standards-compat-2`</sub>

### 3. Every probe change on every platform must now edit probes_etw.txt

A Linux developer adds an argument to an lwlock probe in probes.d. The build fails on their machine until they also edit probes_etw.txt and choose an ETW keyword, level and field name for a facility they never use. Argument descriptions now live in probes.d, monitoring.sgml and probes_etw.txt.

The commit message makes the consistency check run in meson and make builds without ETW, so the cost falls on all developers while only MSVC Windows users gain (item 4). The check fails loudly, which answers item 9, but it adds a third copy of each argument's type, plus a field name that the file header says is copied by hand from monitoring.sgml. A committer would likely ask whether keyword, level and names could be derived or defaulted, so that probes.d stays the only file to edit.

**Question:** Why can probes_etw.txt not be generated, or given defaults, from probes.d and the existing documentation, so a non-Windows probe change needs no second file?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#1](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com); `maintenance-fit-1`</sub>

### 4. Keyword bits and event fields become a contract with no stated stability rule

An admin saves a logman or wpr profile that filters on keyword mask 0x3 and level 4, as message #1 shows. A later release reorders the PG_KW_* bits or renames a probe argument in probes_etw.txt. The saved profile then captures the wrong events or fields, and the admin finds out only when a trace comes back empty or mislabeled.

The docs promise that the provider name and GUID never change. Capture profiles also depend on the keyword bit values, the levels, the event names and the field names, and the example in message #1 hard-codes the mask 0x3. The proposal does not say whether these stay stable across major versions or may change with probes.d. Without a stated rule, users will treat them as stable, and every later probe change becomes a compatibility question.

**Question:** Which parts of the event schema (keyword bits, levels, event and field names) are promised stable across major versions, and how are users told when they change?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com); `standards-compat-1`</sub>

### 5. No evidence that Windows users want to trace the static probes

The project adds and documents a provider GUID as a contract that "never changes", plus a second probe file that has to be kept up to date. If few Windows users ever start an ETW session for it, the project keeps that contract and file for years for a feature with little use.

Message #1 gives parity with other platforms as the only reason. It cites no field report, no -general or -bugs thread, no second user and no production story. Even on Linux and macOS the static probes need a build option that is off by default, so their user base may be small already. No one has replied in the thread.

**Question:** Which Windows users or support cases needed these probes, what did they use instead, and will the common Windows installers ship with -Detw enabled?

<sub>fixable, level 1b, demand; messages [#1](https://www.postgresql.org/message-id/31c8eade-54a8-4283-be36-ac3a4fbb9c79%40gmail.com); `demand-1`</sub>

## Questions for the author

1. Who may enable the provider, and should the query text fields or the auto default be gated, given that Performance Log Users members can capture every session's queries?
2. How does a WPA or tracerpt user read query text from a WIN1252 or EUC_JP database, and what happens to a multibyte character cut at 32kB?
3. Can the keyword, level and field names be generated or defaulted from probes.d, so that a probe change on a non-Windows platform needs no edit to probes_etw.txt?
4. Which parts of the event schema (keyword bits, levels, event names, field names) stay stable across major versions, besides the provider name and GUID?
5. Which Windows users or support cases needed these probes, and what did they use instead?

## Touch points

- **ABI / extension API:** The probe macros in probes.h now expand to TraceLogging calls on a provider handle, which Windows extensions that use the macros would need.
- **security / privileges:** QUERY events carry query text, and message #1 says membership in Performance Log Users is enough to capture them.
- **backpatchability:** see objection standards-compat-1: Keyword bits and event fields become a contract with no stated stability rule
- **maintenance burden ("who maintains this in 10 years"):** The patch adds probes_etw.txt, a build check on every platform, a permanent provider GUID and a dependency on a Windows SDK header.
- **performance on the hot path:** Probe sites in lwlock and buffer code become live enabled-checks in default MSVC builds.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** see objection demand-1: No evidence that Windows users want to trace the static probes

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the Windows MSVC build and process startup under EXEC_BACKEND, and who can test with wpr or logman.
- Priority: fix design first; audience niche; review cost medium. The benefit goes to Windows MSVC users who trace, and the main patch is about 32 KiB across the build system, postmaster.c and miscinit.c. The security default and the string encoding should be settled before code review, since both change what the provider emits.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
