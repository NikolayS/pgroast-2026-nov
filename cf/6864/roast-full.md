# Design pre-review: Add explicit warnings about unsafe OAuth trace output for libpq

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6864](https://commitfest.postgresql.org/patch/6864/) (Needs review); authors: Zsolt Parragi (zsolt.parragi)
- Thread: [Add explicit warnings about unsafe OAuth trace output for libpq](https://www.postgresql.org/message-id/flat/CAN4CZFOUG9csxguH_%2Bcz%3DnQ8HQVKLcVvr%3DqJSbga%2Btj-3FZhLA%40mail.gmail.com), started 2026-04-07
- Messages read: 4 of 4
- Patch prose read: 0001-libpq-oauth-Warn-when-PGOAUTHDEBUG-trace-may-expose-.patch, rel18-0001-libpq-oauth-Warn-when-PGOAUTHDEBUG-trace-may-expose-nocfbot.patch, rel18-0001-libpq-oauth-Warn-when-PGOAUTHDEBUG-trace-may-expose-.patch, 0001-libpq-oauth-Warn-when-PGOAUTHDEBUG-trace-may-expose-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium)

No objection survived at levels 1a, 1b or 2. The problem is stated: PGOAUTHDEBUG trace output contains bearer tokens and client secrets, and the output carries no warning. The change is two stderr messages in one file, seen only by users who set the unsafe trace flag, so it costs other users nothing. Confidence is medium because the wording and the exit paths could not be checked without the code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The wording of the warnings, the end warning on early-abort paths and the stderr path they use could not be checked without the code. The pass at levels 1-2 is provisional on those points.

## The proposal, restated

**Problem.** With PGOAUTHDEBUG trace enabled, libpq-oauth prints raw HTTP traffic to stderr, including bearer tokens and client secrets. The documentation already calls this unsafe, but the output itself carries no warning. Users who paste the long output into bug reports or mailing list posts may not notice that it contains secrets.

**Proposed design.** libpq-oauth prints a warning at the start of the OAuth flow, right after it parses the debug flags, saying trace logging is active and includes secrets. At the end of the flow, on success or failure, it prints a reminder that the trace output above may contain secrets and should not be shared. The author also offers a Postgres 18 version, where the trigger is PGOAUTHDEBUG=UNSAFE.

**User-visible surface.**
- Two new warning messages on stderr that bracket the libcurl verbose trace when PGOAUTHDEBUG trace (master) or PGOAUTHDEBUG=UNSAFE (18) is set.

**What the author claims.**
- The warnings mark the start and end of the sensitive output, however much output curl produces in between.
- The change is useful to backport to 18, which needs a separate patch because 19 changed the debug flags.

**Thread state.** Daniel Gustafsson supports the idea and proposes to go ahead unless someone objects; nobody else replied. The author has rebased the same patches twice, and the entry is in its third CommitFest.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-04-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

No objection held up strongly enough in the counter-check to list here.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The warnings concern bearer tokens and client secrets that PGOAUTHDEBUG trace prints to stderr. |
| backpatchability | **yes** | The author offers a separate patch for 18, where the trigger is PGOAUTHDEBUG=UNSAFE; the thread has no backpatch decision. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Does the end warning print on every exit path, including an early abort of the flow or a failure before the first HTTP request?
2. What is the exact wording of the two warnings, and is it the same on 18 and on master?
3. Do the warnings go to the same stderr stream as the libcurl trace, so that a redirect captures them together with the secrets?
4. Is the 18 patch meant for backpatching, given that it adds new stderr output in a stable branch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. The owner should know the libpq-oauth client flow in oauth-curl.c. According to the brief, Daniel Gustafsson supports the idea in message #4 and proposes to go ahead unless someone objects.
- **Liability:** Two stderr messages in one file, kept in step with the debug flag parsing. With a backpatch, the owner also signs for new stderr output on 18 for users of PGOAUTHDEBUG=UNSAFE.
- **Author follow-up:** The author rebased the patches twice, in messages #2 and #3, and moved the entry through three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** Only users who enable unsafe OAuth trace debugging see the change, and for them it lowers the risk of sharing secrets. The patch is under 3 KiB in one file, and the brief reports support in message #4.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). The precedent search found no earlier proposal with the same or a similar idea, so no prior rejection reason applies. Message #1 points to an earlier thread on splitting the debug flags; according to message #1, the warning idea was moved out of that thread into this one, so it was not rejected there.
- standards-compat: 0 objection(s). The proposal adds two stderr messages only when a user has set the PGOAUTHDEBUG unsafe trace mode, so it introduces no new syntax, GUC, psql convention, dump content or protocol change. Adding text to opt-in debug output on 18 is unlikely to break a working setup, and the exact wording cannot be checked without the code.
- replication-ondisk: 0 objection(s). The patch adds two stderr messages in libpq-oauth's client-side flow in oauth-curl.c. It does not touch on-disk format, WAL, replication, catalogs, the wire protocol or a public libpq struct, so this angle finds nothing to object to. The ABI impact of the 18 backport could not be checked without the code, but the commit message describes only output changes.
- security: 0 objection(s). The patch adds two stderr warnings around trace output that the user already opted into with the UNSAFE prefix, and it changes no privilege, default or exposure. It reduces the chance that secrets get shared, and I found no failure scenario that would justify an objection from the security angle.
- maintenance-fit: 0 objection(s). No maintenance or fit objection holds up. The change is two stderr messages in one file, src/interfaces/libpq-oauth/oauth-curl.c, and they print only when an explicit UNSAFE debug flag is set. That means no hot-path cost for anyone else, no new API, GUC or catalog state, and nothing later features must keep in sync. Daniel Gustafsson, who knows the area, supports it in message #4. Backpatching to 18 changes stderr output only for users who opted into unsafe tracing. Not checked without the code: whether the end warning prints on early abort.
- already-exists: 0 objection(s). From the existing-feature angle there is no alternative to name. The documentation already warns about PGOAUTHDEBUG trace output, and the patch puts that warning into the output itself, where users who skip the docs will see it. No extension or other core mechanism does this, so the patch adds no second way to do an existing job.
- demand: 0 objection(s). Demand is no fit for this patch. It adds two stderr lines in one file, and only users who opt in to an UNSAFE debug flag see them, so the cost to other users is close to zero. A committer who knows the area supports it in message #4, and the idea came from an earlier review thread mentioned in message #1.

### Precedents retrieved

- P1: [OAuth Followup Patches](https://commitfest.postgresql.org/patch/5635/) (cf, Withdrawn, 2025-06-26) - judged not relevant; reason summarised from its thread
- P2: [Missing dependency on oauth_validator test](https://commitfest.postgresql.org/patch/6407/) (cf, Withdrawn, 2026-01-19) - judged not relevant; reason summarised from its thread
- P3: [Revert "oauth: Add unit tests for multiplexer handling"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ebaaf386adb133010c2024256521b993c5e53e98) (revert, Reverted, 2025-08-08) - judged not relevant
- P4: [Improve OAuth discovery logging \[accidental duplicate\]](https://commitfest.postgresql.org/patch/6536/) (cf, Withdrawn, 2026-02-26) - judged not relevant; reason summarised from its thread
- P5: [libpq trace log](https://commitfest.postgresql.org/patch/1854/) (cf, Returned with feedback, 2019-09-30) - judged not relevant; reason summarised from its thread
- P6: [User functions for building SCRAM secrets](https://commitfest.postgresql.org/patch/3988/) (cf, Returned with feedback, 2024-01-27) - judged not relevant; reason summarised from its thread
- P7: [pg_tracing](https://commitfest.postgresql.org/patch/4456/) (cf, Withdrawn, 2024-07-10) - judged not relevant; reason summarised from its thread
- P8: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P9: [Report tranche ID and LWLock pointer in LWLock static tracepoints](https://commitfest.postgresql.org/patch/2927/) (cf, Returned with feedback, 2021-04-07) - judged not relevant; reason summarised from its thread
- P10: [Revert "Temporarily install debugging in partition_prune test"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ec1a405adc24bc20523eef83ccc85232ec23eaa9) (revert, Reverted, 2024-05-14) - judged not relevant
- P11: [Use role name "system_user" instead of "user" for unsafe_tests](https://commitfest.postgresql.org/patch/4271/) (cf, Withdrawn, 2023-04-13) - judged not relevant; reason summarised from its thread
- P12: [Infrastructure changes to make the code less sensitive to transaction id bit width](https://commitfest.postgresql.org/patch/2960/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P13: [Revert removal of trace_userlocks, because userlocks aren't gone.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=71b2b657c034743f81ae7906d7b0fcc8e9a0a6a3) (revert, Reverted, 2011-11-10) - judged not relevant
- P14: [create index failed. PARALLEL UNSAFE function does not prevent parallel index build](https://commitfest.postgresql.org/patch/4860/) (cf, Rejected, 2024-03-13) - judged not relevant
- P15: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P16: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P17: [libpq compression](https://commitfest.postgresql.org/patch/2773/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P18: [libpq compression](https://commitfest.postgresql.org/patch/3499/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P19: [Revert GetTransactionSnapshot() to return historic snapshot during LR](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=50f770c3d92cfb2d62724d8c810d37c1f6249a11) (revert, Reverted, 2025-08-22) - judged not relevant
- P20: [Libpq Compression](https://commitfest.postgresql.org/patch/4746/) (cf, Withdrawn, 2024-05-14) - judged not relevant
- P21: [Make it possible to prevent some unsafe operations via a contrib hook.](https://commitfest.postgresql.org/patch/682/) (cf, Returned with feedback, 2016-10-01) - judged not relevant
- P22: [Add force flag while doing pg_restore to force dropping database in case of existing connections](https://commitfest.postgresql.org/patch/4485/) (cf, Returned with feedback, 2024-01-14) - judged not relevant
- P23: [Revert "Use group updates when setting transaction status in clog."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=390811750d1647ed81a462da3af7f1fd66166e95) (revert, Reverted, 2017-03-10) - judged not relevant
- P24: [Row Level Security, leakproof-ness, and performance implications](https://commitfest.postgresql.org/patch/2038/) (cf, Rejected, 2019-03-19) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 106 s; cost reported by the CLI: $1.22
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 1 from cache, 7 summarised now, 0 failed; quotes: 6 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 40 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:06:14.241Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
