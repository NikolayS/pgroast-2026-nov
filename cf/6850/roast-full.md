# Design pre-review: Don't consider FATAL errors "abnormal server termination"

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6850](https://commitfest.postgresql.org/patch/6850/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [libpq maligning postgres stability](https://www.postgresql.org/message-id/flat/up7whuxn263agr7whdzzoak7mo7ywryc7z7z3ejdvgbbke4vji%403puckkgm4ide), started 2025-03-27
- Messages read: 7 of 7
- Patch prose read: v1-0001-libpq-Consider-a-connection-with-a-FATAL-error-to.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: four participants agreed in 2025 that the appended text misleads, and demand is shown in message #1. Two fixable points remain. standards-compat-1 asks what clients that match the appended string should use instead. replication-ondisk-1 asks which libpq callers depend on the old PQstatus() timing, since v2 had to change postgres_fdw. Both need an answer from someone who knows libpq connection state, but neither requires a different proposal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v2 commit message and the postgres_fdw change are not in the material, so the effect on other libpq callers could not be judged. Behaviour during startup, COPY and pipeline mode needs the code.

## The proposal, restated

**Problem.** libpq appends "server closed the connection unexpectedly / This probably means the server terminated abnormally" even after the server sent a FATAL, such as after pg_terminate_backend(). The hint misleads users into thinking Postgres crashed. It also stops clients from using PQresultErrorField on the final PGresult.

**Proposed design.** When libpq receives a FATAL or PANIC error, it sets the connection status to CONNECTION_BAD. libpq then treats the following socket close as expected and does not append the "unexpectedly" text. v2 also changes postgres_fdw and fixes tests, and the commit message is said to explain the postgres_fdw change.

**User-visible surface.**
- libpq no longer appends "server closed the connection unexpectedly" and the abnormal-termination hint after a FATAL or PANIC error.
- PQstatus() returns CONNECTION_BAD once a FATAL or PANIC error has arrived, possibly before the socket closes.
- The last PGresult after a FATAL holds the server error fields, so PQresultErrorField can read them.
- postgres_fdw behavior changes in v2, but the material does not say how.

**What the author claims.**
- The fix is small: mark the connection CONNECTION_BAD when a FATAL or PANIC arrives.
- The appended text is wrong, because the server told the client it was closing the connection.
- The appended text makes errors harder for clients to parse.
- v2 fixes the test failures that cfbot found in v1.
- The change targets PG20.

**Thread state.** Andres Freund, Robert Haas, Christoph Berg and Bruce Momjian agreed in 2025 that the message misleads, and Berg warned that much software parses its first line. Freund reported v1 test failures, and nobody has replied to v2, which is in its third CommitFest.

## 2. Prior attempts

### Revert commits 290acac92b and 8a7e9e9dad. (related)

- **Link:** [https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d)
- **Thread:** [https://www.postgresql.org/message-id/flat/E1iaD8h-0004us-K9@gemulon.postgresql.org](https://www.postgresql.org/message-id/flat/E1iaD8h-0004us-K9@gemulon.postgresql.org), [https://www.postgresql.org/message-id/flat/CAP_rwwmLJJbn70vLOZFpxGw3XD7nLB_7+NKz46H5EOO2k5H7OQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAP_rwwmLJJbn70vLOZFpxGw3XD7nLB_7+NKz46H5EOO2k5H7OQ@mail.gmail.com)
- **Date:** decided 2019-11-30; started 2019-11-28
- **Outcome:** Reverted (revert)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (as recorded in the corpus):** This commit revert the commits to add a test case that tests the 'force' option when there is an active backend connected to the database being dropped. This feature internally sends SIGTERM to all the backends connected to the database being dropped and then the same is reported to the client. We found that on Windows, the client end of the socket is not able to read the data once we close the socket in the server which leads to loss of error message which is not what we expect. We also observed similar behavior in other cases like pg_terminate_backend(), pg_ctl kill TERM <pid>. There are probably a few others like that. The fix for this requires further study.
- **Reason (quote):** "We found that on Windows, the client end of the socket is not able to read the data once we close the socket in the server which leads to loss of error message which is not what we expect."
- **What is different this time:** P1 removed tests that needed the client to read the FATAL after pg_terminate_backend() or a SIGTERM, because Windows can drop that message. This proposal changes libpq only when the FATAL does arrive, so where Windows drops it the old text stays.


## 3. Objections a reviewer is likely to raise

### 1. Clients that match the appended string lose reconnect detection

fixable · level 1b, rubric item 6 · desirability, wire protocol, backpatchability · `standards-compat-1`

An application uses a framework that detects a lost connection by matching "server closed the connection unexpectedly" and then reconnects. After its libpq is upgraded, a session ended by pg_terminate_backend() or a restart returns only the FATAL text. The framework treats it as an ordinary query error and does not retry, and operators find out from failed requests after a routine failover.

Message #3 lists several packages that parse this exact string to detect lost connections, and warns against changing it. The patch removes the string in the FATAL case, which is the usual case after an administrator command or a shutdown. The commit message argues the text is wrong, but it does not weigh what existing parsers lose or offer them a stable replacement. A committer would likely ask for this to be called out as a compatibility change and for the replacement to be documented.

Thread: [#3 (Christoph Berg, 2025-03-27)](https://www.postgresql.org/message-id/Z-V2v_yqewKY51BS%40msg.df7cb.de), [#5 ("Jelte Fennema-Nio", 2026-05-26)](https://www.postgresql.org/message-id/DISFEF23Y30A.3H6YMBMPYVF8I%40jeltef.nl), [#7 ("Jelte Fennema-Nio", 2026-06-11)](https://www.postgresql.org/message-id/DJ60NPKYB9L9.3I3SS9I2MBL00%40jeltef.nl)

**Question:** How should applications that match this string today detect an admin-terminated connection after the change, and will the release notes and docs say so?

Counter-check: held. Message #3 lists software that matches this exact line, and the patch removes the line in the FATAL case. The thread does not answer the warning or say that these clients have another signal, such as PQstatus() or the SQLSTATE.

### 2. Earlier CONNECTION_BAD changes libpq API behavior for existing callers

fixable · level 2, rubric item 11 · ABI / extension API, WAL / replication / standby, backpatchability · `replication-ondisk-1`

An extension, or libpqwalreceiver in a logical apply worker, receives a FATAL and then checks PQstatus() or keeps calling PQgetResult() as it did before. With the patch, PQstatus() reports CONNECTION_BAD before the socket closes, so the old code path changes. The extension author finds out when the extension's tests fail or its error reporting changes on PG20.

v2 had to change postgres_fdw (message #7), so at least one in-tree libpq consumer depended on the old ordering. The material does not say what that change is, or whether dblink, libpqwalreceiver and the logical replication workers were checked. Out-of-tree code built on libpq would need the same change, with no compile-time signal. The commit message, the only place where the postgres_fdw change is explained, is not in the material.

Thread: [#5 ("Jelte Fennema-Nio", 2026-05-26)](https://www.postgresql.org/message-id/DISFEF23Y30A.3H6YMBMPYVF8I%40jeltef.nl), [#7 ("Jelte Fennema-Nio", 2026-06-11)](https://www.postgresql.org/message-id/DJ60NPKYB9L9.3I3SS9I2MBL00%40jeltef.nl)

**Question:** Which caller pattern did postgres_fdw rely on, and have dblink, libpqwalreceiver and the logical replication workers been checked for the same pattern?

Counter-check: held. Message #7 confirms that postgres_fdw needed a change, so the new PQstatus() timing is visible to callers. The v2 commit message that explains the change is not in the material, and nobody has reviewed v2, so the question about other callers stays open.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | libpqwalreceiver and logical replication workers are libpq callers that see CONNECTION_BAD earlier after a FATAL. |
| wire protocol | **yes** | libpq reacts differently to an ErrorResponse with FATAL or PANIC severity; no message format changes. |
| catalog changes | no |  |
| ABI / extension API | **yes** | PQstatus() returns CONNECTION_BAD before the socket closes, and v2 had to change postgres_fdw for it. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The entry is tagged Bugfix but changes error text and PQstatus() timing that deployed clients depend on; the author targets PG20 only. |
| maintenance burden ("who maintains this in 10 years") | no |  |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. How should applications that match "server closed the connection unexpectedly" detect an admin-terminated connection after the change, and will the docs and release notes say so?
2. What did postgres_fdw assume about PQstatus() after a FATAL, and have dblink, libpqwalreceiver and the logical replication workers been checked for the same assumption?
3. How does libpq behave when the FATAL arrives during connection startup, inside COPY, or in pipeline mode?
4. Does v2 pass the full test suite on Windows, where the client can lose the server's final FATAL message?
5. Is a change to the message and hint for real unexpected closes left for a separate patch?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund started the thread and reported the v1 test failures, but nobody has replied to v2. The owner needs to know libpq connection state handling and its in-tree callers.
- **Liability:** The owner signs for a behaviour change in every libpq-based client: different error text after a FATAL and earlier CONNECTION_BAD. Reports from drivers, poolers and frameworks can arrive for several releases.
- **Author follow-up:** The author posted v2 after the v1 test failures and has moved the entry through three CommitFests.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The goal has agreement and v1 touches one libpq file, but v2 has had no reply across three CommitFests. The v2 patch, including its postgres_fdw change, was not read, so the review cost is an estimate.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` PQstatus change that broke postgres_fdw will also hit other clients. Refuted (not specific): Duplicate of replication-ondisk-1, which makes the same point about PQstatus() timing and also names the in-tree callers to check.
- `maintenance-fit-1` Unexplained postgres_fdw change suggests other libpq clients need it too. Refuted (not specific): Duplicate of replication-ondisk-1.
- `maintenance-fit-2` String-matching clients lose their lost-connection signal after FATAL. Refuted (not specific): Duplicate of standards-compat-1, which rests on the same message #3 evidence and asks for a documented replacement signal.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent rejected this idea. P1 shows that on Windows the client can lose the FATAL; that limits the patch's reach and may make tests that expect the new output flaky, but it is no reason to reject the patch. The 2023 discussion Bruce Momjian cites in message #4 is reported only as having reached no conclusion.
- standards-compat: 2 objection(s). Two objections raised; both are fixable with documentation and a clear statement of the PQstatus contract.
- replication-ondisk: 1 objection(s). The patch changes no on-disk format, WAL or catalog, and adds no new protocol message. It only changes how libpq reacts to an existing ErrorResponse, so the one objection here is about the libpq API seen by callers, including in-tree replication code.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: libpq drops the appended text and sets CONNECTION_BAD once the server has sent FATAL or PANIC. Who can send a FATAL, for example through pg_terminate_backend(), does not change, and nothing new becomes visible. Without the code I could not check how a FATAL during startup or authentication is handled.
- maintenance-fit: 2 objection(s). The fix is small, stays inside libpq, and has no hot-path or extension-fit cost. The remaining concerns are about compatibility for existing libpq clients.
- already-exists: 0 objection(s). No existing libpq option, error verbosity setting or documented idiom stops libpq from appending the "unexpectedly" text after a FATAL or from overwriting the error fields of the final PGresult. This is internal libpq connection-state handling, so an extension or external tool cannot do it, and the patch fixes the existing mechanism instead of adding a second one.
- demand: 0 objection(s). Demand is shown: Andres Freund reports fielding worried user questions about the message (message #1), Haas, Berg and Momjian agree it misleads (messages #2-#4), and the author has hit it repeatedly (message #5). The cost falls on clients that match the appended text after a FATAL, which is a compatibility question for another angle.

### Precedents retrieved

- P1: [Revert commits 290acac92b and 8a7e9e9dad.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d) (revert, Reverted, 2019-11-30) - used in the review
- P2: [Move libpq's write_failed mechanism down to pqsecure_raw_write().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faa189c932d51945b2285e277128b0f26b96afdd) (revert, Partially reverted, 2022-02-12) - used in the review
- P3: [libpq should append error messages, not overwrite](https://commitfest.postgresql.org/patch/1752/) (cf, Returned with feedback, 2018-10-01) - judged not relevant; reason summarised from its thread
- P4: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=21ef4d4d897563adb2f7920ad53b734950f1e0a4) (revert, Reverted, 2024-01-23) - judged not relevant
- P5: [Allow specifying multiple hosts in libpq connect options](https://commitfest.postgresql.org/patch/222/) (cf, Returned with feedback, 2015-08-25) - judged not relevant; reason summarised from its thread
- P6: [Revert ill-conceived patch that made elog(FATAL) the same as elog(ERROR) followed by seeing EOF from client.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98c120a2037391ad1d475b7e3566505ede36cb77) (revert, Reverted, 2004-07-28) - judged not relevant
- P7: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant; reason summarised from its thread
- P8: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P9: [Improving connection scalability](https://commitfest.postgresql.org/patch/3651/) (cf, Rejected, 2022-08-01) - judged not relevant; reason summarised from its thread
- P10: [Make all Perl warnings fatal](https://commitfest.postgresql.org/patch/4501/) (cf, Returned with feedback, 2023-09-12) - judged not relevant; reason summarised from its thread
- P11: [postgres_fdw - make cached connection functions tests meaningful with debug_invalidate_system_caches_always](https://commitfest.postgresql.org/patch/3112/) (cf, Rejected, 2021-07-21) - judged not relevant; reason summarised from its thread
- P12: [postgres_fdw: Handle boolean comparison predicates](https://commitfest.postgresql.org/patch/3144/) (cf, Returned with feedback, 2021-10-01) - judged not relevant; reason summarised from its thread
- P13: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - judged not relevant
- P14: [Revert "Close stdin where it's not needed in TestLib.pm procedures"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca266a069a20c32a8f0a1df982a5a67d9483bcb3) (revert, Reverted, 2019-11-27) - judged not relevant
- P15: [Fix collation handling in postgres_fdw](https://commitfest.postgresql.org/patch/2685/) (cf, Returned with feedback, 2021-09-25) - judged not relevant
- P16: [Optional message to signalled process in pg_terminate_backend()/pg_cancel_backend()](https://commitfest.postgresql.org/patch/1174/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P17: [HINTing on UPDATE foo SET foo.bar = ..](https://commitfest.postgresql.org/patch/415/) (cf, Returned with feedback, 2016-03-09) - judged not relevant
- P18: [idle-in-transaction timeout error does not give a hint](https://commitfest.postgresql.org/patch/1892/) (cf, Withdrawn, 2019-03-31) - judged not relevant
- P19: [Revert changes for SSL compression in libpq](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0ba71107efeeccde9158f47118f95043afdca0bb) (revert, Reverted, 2021-03-10) - judged not relevant
- P20: [Add Change Badges to documentation](https://commitfest.postgresql.org/patch/2309/) (cf, Returned with feedback, 2019-11-25) - judged not relevant
- P21: [Full support for index LP_DEAD hint bits on standby](https://commitfest.postgresql.org/patch/2947/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P22: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P23: [Detection and resolution of logical replication conflict](https://commitfest.postgresql.org/patch/5021/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P24: [Mention column name in error messages](https://commitfest.postgresql.org/patch/324/) (cf, Returned with feedback, 2015-12-22) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 125 s; cost reported by the CLI: $1.39
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 2 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:05:21.713Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
