# Design pre-review: Don't use the deprecated and insecure PQcancel in our frontend tools anymore

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6314](https://commitfest.postgresql.org/patch/6314/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [Don't use the deprecated and insecure PQcancel in our frontend tools anymore](https://www.postgresql.org/message-id/flat/DEY0N7FS8NCU.1F7QXGG37MQ09%40jeltef.nl), started 2025-12-14
- Messages read: 32 of 32
- Patch prose read: v13-0001-psql-Stop-scripts-on-Ctrl-C-while-waiting-for-CO.patch, v13-0002-psql-Replace-cancel_pressed-with-CancelRequested.patch, v13-0003-fe_utils-Simplify-cancel-logic-in-wait_on_slots.patch, v13-0004-Move-Windows-pthread-compatibility-functions-to-.patch, v13-0005-Don-t-use-deprecated-and-insecure-PQcancel-psql-.patch, v1-0001-Move-Windows-pthread-compatibility-functions-to-s.patch, v1-0002-Don-t-use-the-deprecated-and-insecure-PQcancel-in.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No surviving objection questions the goal or the overall design. The only surviving objection, prior-attempts-1, is a fixable portability point: v13-0003 uses pselect() behind an existence check, and [P12] records that some platforms shipped a pselect() that did not work. A committer who knows the frontend signal and portability code can settle that, and the 1 second select() fallback already exists.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the review could not check thread safety of siglongjmp() out of input waits, the wait in SetCancelConn() and ResetCancelConn() for an in-flight cancel, or the pselect() signal masking. Only 3 of 32 messages were visible, so later answers may exist.

## The proposal, restated

**Problem.** psql, pg_dump, pgbench, vacuumdb and other frontend tools still send cancel requests with PQcancel, which does not use encryption. So the cancel key travels in clear text even when the session uses SSL. The author wants all tools using fe_utils cancel handling to stop doing that.

**Proposed design.** The tools switch to the newer libpq cancel API (61461a300), which uses the same encryption settings as the cancelled connection. That API is not signal-safe, so the SIGINT handler now wakes a dedicated cancel thread through a self-pipe, as Windows already does. SetCancelConn() and ResetCancelConn() wait for any in-flight cancel, and wait_on_slots() uses pselect() or a 1 second select() poll.

**User-visible surface.**
- Cancel requests from psql, pg_dump, pg_restore, pgbench, pg_amcheck, clusterdb, reindexdb and vacuumdb now use the connection's SSL/GSS encryption.
- psql -f scripts now stop with exit code 3 on Ctrl+C during \copy ... from pstdin, \prompt or \password input (committed as 0001).
- With a stuck cancel connection, the psql prompt stays blocked until the cancel request finishes or times out.
- fe_utils cancel.h API changes, including a new setup_cancel_handler() taking a signal callback and a thread callback.
- New configure and meson check for pselect().

**What the author claims.**
- PQcancel is insecure because it sends the cancel request without encryption.
- The design makes cancel handling more uniform between Windows and Unix, and does not need changes at call sites.
- The remaining race, where a late cancel hits a later query, comes from the cancel protocol and is unlikely in interactive use.
- Backporting to PG17 and PG18 would be desirable as a security fix, but may be too large.

**Thread state.** Heikki Linnakangas reviewed in depth, Zsolt Parragi, Anthonin Bonnefoy and Michael Paquier commented, and Peter Eisentraut committed v13-0001 and is reviewing the rest. Backpatching and Heikki's design ideas, such as cancel inside libpq and mutually exclusive callbacks, have no resolution.

## 2. Prior attempts

### Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop." (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0)
- **Thread:** [https://www.postgresql.org/message-id/flat/9696.1493072081@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/9696.1493072081@sss.pgh.pa.us), [https://www.postgresql.org/message-id/flat/9205.1492833041@sss.pgh.pa.us](https://www.postgresql.org/message-id/flat/9205.1492833041@sss.pgh.pa.us)
- **Date:** decided 2017-04-24; started 2017-04-24
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This reverts commit 81069a9efc5a374dd39874a161f456f0fb3afba4. Buildfarm results suggest that some platforms have versions of pselect(2) that are not merely non-atomic, but flat out non-functional. Revert the use-pselect patch to confirm this diagnosis (and exclude the no-SA_RESTART patch as the source of trouble). If it's so, we should probably look into blacklisting specific platforms that have broken pselect.
- **Reason (quote):** "Buildfarm results suggest that some platforms have versions of pselect(2) that are not merely non-atomic, but flat out non-functional."
- **What is different this time:** This uses pselect() only in frontend wait_on_slots(), not in the postmaster, and falls back to select() with a 1 second timeout. The thread does not discuss how a broken pselect() would be detected.

### pgbench: allow to cancel queries during benchmark (related)

- **Link:** [https://commitfest.postgresql.org/patch/4399/](https://commitfest.postgresql.org/patch/4399/)
- **Thread:** [https://www.postgresql.org/message-id/flat/20230626224628.61f372f25cde3747f164a4c6@sraoss.co.jp](https://www.postgresql.org/message-id/flat/20230626224628.61f372f25cde3747f164a4c6@sraoss.co.jp)
- **Date:** decided 2024-10-07; started 2023-06-26
- **Outcome:** Returned with feedback (cf)
- **Who objected:** Fabien COELHO, Alvaro Herrera (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Fabien COELHO said thread #0 cancelling clients that other threads manage can cause race conditions, and asked that each thread cancel its own clients. In message #19 the author found two more problems: thread #0 cannot cancel anything once it leaves its loop, and with -C each thread recreates its PGcancel objects while thread #0 reads them. Alvaro Herrera then said commit 61461a300c1c, which deprecated PQcancel, requires the patch to be reworked. The author planned a redesign, but the visible messages show no new version.
- **Reason (quote):** "Due to commit 61461a300c1c, this patch needs to be reworked." ([message](https://www.postgresql.org/message-id/202403301335.jcac6f3tkf2f%40alvherre.pgsql))
- **What is different this time:** That patch added Ctrl+C cancel for all pgbench clients and stopped because it had to move to the new cancel API. This proposal makes that move for fe_utils users but keeps one global cancel connection.


## 3. Objections a reviewer is likely to raise

### 1. pselect() was reverted before because some platforms ship a broken one

fixable · level 2, rubric item 7 · prior rejection, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

On a platform whose pselect() exists but does not work, configure detects it and wait_on_slots() uses it. Then vacuumdb -j or reindexdb -j can miss Ctrl+C or fail while it waits on its slots. The buildfarm or a user on that platform finds out after commit.

[P12] records that pselect() in the postmaster loop was reverted because some platforms' versions "are not merely non-atomic, but flat out non-functional". v13-0003 makes the first use of pselect() since then, behind a plain configure check (message #22 notes it is unused today). The thread does not say whether those platforms are still supported or how a broken pselect() would be detected. The 1 second select() fallback already exists, so a narrower or opt-in use is a possible way out.

**Precedents.**
- [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (Reverted, 2017-04-24); thread: [#1](https://www.postgresql.org/message-id/flat/9696.1493072081@sss.pgh.pa.us), [#2](https://www.postgresql.org/message-id/flat/9205.1492833041@sss.pgh.pa.us)

Thread: [#22 (Heikki Linnakangas, 2026-07-07)](https://www.postgresql.org/message-id/a4c6998e-d026-4eed-bb90-c9ad121aaed0%40iki.fi), [#27 ("Jelte Fennema-Nio", 2026-09-02)](https://www.postgresql.org/message-id/DL519QREJ4P1.2Z9UAEYVCTNQ2%40jeltef.nl)

**Question:** Which platforms had the broken pselect() behind the earlier revert, and are any of them still supported or in the buildfarm today?

Counter-check: held. [P12] is about the postmaster loop, but its recorded reason, a pselect() that does not work on some platforms, applies to any caller. The thread does not say whether those platforms are still supported, and message #27 chose pselect() without discussing this.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | **yes** | The tools now send cancel requests over an encrypted connection through the existing libpq cancel API; the protocol itself does not change. |
| catalog changes | no |  |
| ABI / extension API | **yes** | The fe_utils cancel.h API changes, including a new setup_cancel_handler() with two callbacks, and pthread-win32.c moves to src/port. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | **yes** | psql -f scripts exit with code 3 on Ctrl+C at input prompts, and the prompt can stay blocked on a stuck cancel connection. |
| security / privileges | **yes** | The cancel key no longer travels in clear text when the session uses SSL or GSS. |
| backpatchability | **yes** | Message #1 asks about a backport to PG17 and PG18 as a security fix; the thread has not settled it. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Every frontend tool gains a cancel thread, a self-pipe and a new configure and meson check for pselect(). |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which platforms had the broken pselect() behind the revert in [P12], and are any of them still supported or in the buildfarm today?
2. What do the tools do when the cancel thread cannot be created, or on a platform without threads?
3. Is a second call to setup_cancel_handler() now guarded against, as raised in review?
4. Has cancel through a bad network been tested for tools other than psql, such as pg_dump with parallel workers?
5. Is the plan master-only, or is a backport to PG17 and PG18 still wanted?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Peter Eisentraut committed v13-0001 and is listed as reviewer; Heikki Linnakangas reviewed the design in depth. The owner needs to know frontend signal handling, threads and platform portability.
- **Liability:** The owner signs for Ctrl+C behaviour in psql, pg_dump, pgbench and the scripts on every supported platform, including the signal handler, self-pipe, cancel thread and pselect() path, for as long as these tools exist.
- **Author follow-up:** The author posted 13 versions over about 11 months, rebased repeatedly and answered review, which suggests continued follow-up.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** most users
- **Maintenance burden:** medium
- **Review cost:** high
- **Why:** Every SSL or GSS user of the frontend tools is affected, and committers are already engaged with one part committed. The remaining series changes signal and thread handling in many tools, so review is costly and the new cancel path needs ongoing care.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Cancel can now fail where the plaintext cancel used to succeed. Refuted (already handled): Message #9 shows psql reporting a failed cancel with "Could not send cancel request" while the session continues. A fallback to a cleartext cancel would undo the purpose of the patch, and the handshake rule comes from the libpq API already committed in 61461a300.
- `standards-compat-2` Backport to 17 and 18 would change client behaviour in a minor release. Refuted (already handled): Message #1 already calls a backport probably unrealistic and only asks for opinions, and no one in the thread supports one. The scenario needs a backport that nobody has proposed in concrete form, and the commit to master does not depend on it.
- `maintenance-fit-1` New pselect() dependency repeats a reverted approach without a guard. Refuted (not specific): Duplicate of prior-attempts-1, which states the same pselect() concern with the same precedent and a more focused question.
- `maintenance-fit-2` Thread-based cancel path has no automated test. Refuted (wrong facts): psql already has a TAP test, 020_cancel.pl, that sends SIGINT to a running psql, so the basic cancel path through the new thread would be exercised on Unix. Wider test coverage is a level 4 matter, which the rubric puts out of scope.
- `maintenance-fit-3` Reshaped cancel.h API still fits only one connection per process. Refuted (already handled): The single global cancelConn exists in cancel.c today, so the patch keeps existing practice; message #4 says it is fine for current callers and message #5 answers the scope question. [P4] failed on pgbench thread races and does not show this API is at fault.

### Review angles

- prior-attempts: 1 objection(s). No earlier attempt at the same idea is in the list. The only recorded reason that still applies is the pselect() portability revert.
- standards-compat: 2 objection(s). From the standards angle, the series adds no SQL syntax, GUC, catalog object or pg_dump output. The committed exit-code change was discussed and settled in messages #28 to #32.
- replication-ondisk: 0 objection(s). The proposal changes only how frontend tools send a cancel request, using the libpq cancel API that already exists. It adds no on-disk, WAL, catalog or wire protocol state, and the cancel.h change is internal to fe_utils, not an extension ABI. Not checked without code: whether a pooler that accepts TLS connections handles TLS cancel requests the same way.
- security: 0 objection(s). The proposal tightens security: cancel requests from the frontend tools would use the same SSL or GSS encryption as the session, where the cancel key now goes in clear text. It adds no privilege, role, server-side exposure or attack surface that an unprivileged user could reach. The cancel thread and the blocked prompt on a stuck cancel connection affect only the user's own client process. Backpatching to PG17 and PG18 is still open in the thread, but that is a scope decision rather than a security flaw. Without the code, I could not check how the tools behave when the encrypted cancel connection fails where a plain one would have worked, for example through a pooler.
- maintenance-fit: 3 objection(s). Desirability is not in doubt: sending the cancel key in clear text is a real defect, and Heikki Linnakangas and Peter Eisentraut are engaged and own the area.
- already-exists: 0 objection(s). No existing feature, extension or option lets psql, pg_dump or vacuumdb send an encrypted cancel request today, so the need is not met another way. The patch builds on the existing libpq cancel API and the existing Windows cancel thread rather than adding a second mechanism.
- demand: 0 objection(s). The problem applies to every SSL or GSS user of psql, pg_dump, pgbench and the scripts, because these tools send the cancel key in clear text, and users have no workaround. The project already deprecated PQcancel for this reason (61461a300), and [P4] shows that change forcing rework elsewhere. Several committers are engaged, and one part is committed. Demand is not the weak point here.

### Precedents retrieved

- P1: [Issue in postgres_fdw causing unnecessary wait for cancel request reply](https://commitfest.postgresql.org/patch/4329/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P2: [Refactoring of connection with password prompt loop for frontends](https://commitfest.postgresql.org/patch/2248/) (cf, Returned with feedback, 2019-09-27) - judged not relevant; reason summarised from its thread
- P3: [Extend comments on signal handling and cross-reference comments on signal handling functions](https://commitfest.postgresql.org/patch/2951/) (cf, Returned with feedback, 2021-09-27) - judged not relevant; reason summarised from its thread
- P4: [pgbench: allow to cancel queries during benchmark](https://commitfest.postgresql.org/patch/4399/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P5: [minor fix in CancelVirtualTransaction](https://commitfest.postgresql.org/patch/1903/) (cf, Returned with feedback, 2019-02-03) - judged not relevant; reason summarised from its thread
- P6: [Add "\gi file" to psql to allow COPY with variable expansion](https://commitfest.postgresql.org/patch/5674/) (cf, Returned with feedback, 2025-06-17) - judged not relevant; reason summarised from its thread
- P7: [pg_dump: optimize dumpFunc()](https://commitfest.postgresql.org/patch/5147/) (cf, Withdrawn, 2024-08-02) - judged not relevant; reason summarised from its thread
- P8: [Custom signals and handlers for extension](https://commitfest.postgresql.org/patch/1430/) (cf, Returned with feedback, 2020-11-26) - judged not relevant; reason summarised from its thread
- P9: [Allow multiple recursive self-references](https://commitfest.postgresql.org/patch/3046/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P10: [Fix race condition in SSI when reading PredXact->SxactGlobalXmin](https://commitfest.postgresql.org/patch/6145/) (cf, Withdrawn, 2025-10-22) - judged not relevant
- P11: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant
- P12: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (revert, Reverted, 2017-04-24) - used in the review
- P13: [Fix to_char() behaviour with 'WW-D' pattern](https://commitfest.postgresql.org/patch/2388/) (cf, Returned with feedback, 2020-07-10) - judged not relevant
- P14: [IF (NOT) EXISTS in psql-completion](https://commitfest.postgresql.org/patch/518/) (cf, Returned with feedback, 2016-09-28) - judged not relevant
- P15: [Data at rest encryption](https://commitfest.postgresql.org/patch/2104/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P16: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P17: [Fix psql-completion for CREATE INDEX](https://commitfest.postgresql.org/patch/452/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P18: [GUC to disable cancellation of awaiting for synchronous replication](https://commitfest.postgresql.org/patch/2402/) (cf, Returned with feedback, 2021-10-06) - judged not relevant
- P19: [Document efficient self-joins / UPDATE LIMIT techniques.](https://commitfest.postgresql.org/patch/4539/) (cf, Returned with feedback, 2024-02-13) - judged not relevant
- P20: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P21: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1024/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P22: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant
- P23: [Revert copyright script changes to binary *.key files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=344c26915136fa54c4a1bf54074ba15090b32721) (revert, Reverted, 2020-01-14) - judged not relevant
- P24: [Pipelining/batch mode support for libpq](https://commitfest.postgresql.org/patch/1317/) (cf, Returned with feedback, 2018-11-29) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 141 s; cost reported by the CLI: $3.24
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 80 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:49:03.456Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
