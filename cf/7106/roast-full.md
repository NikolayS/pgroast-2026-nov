# Design pre-review: Avoid building stderr message if possible

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7106](https://commitfest.postgresql.org/patch/7106/) (Needs review); authors: ChangAo Chen (chenchangao)
- Thread: [Avoid building stderr message if possible](https://www.postgresql.org/message-id/flat/tencent_A00A556AC153E46588F0680E34ED4C43EF09%40qq.com), started 2026-08-02
- Messages read: 3 of 3
- Patch prose read: v1-0001-Avoid-building-stderr-message-if-possible.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and no level fails. Message #3 confirms that csvlog and jsonlog never read the buffer, and the default stderr setup pays one boolean test. Three fixable points are open: the check repeated at four sites (maintenance-fit-1), the unannounced '%l' change (standards-compat-1), and a benchmark limited to one 32MB statement (demand-1). The author has not answered message #3.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The claim that all four consumers build the buffer first, and that the final pfree is guarded, rests on message #3. The syslog and eventlog paths were not run.

## The proposal, restated

**Problem.** send_message_to_server_log() always builds a stderr-format message buffer, even when log_destination needs none, such as 'csvlog' or 'jsonlog' alone. The wasted formatting costs time on servers that log a lot, for example with log_min_duration_statement = 0 and large statements.

**Proposed design.** The patch moves the stderr-format buffer construction into a new function, build_stderr_message(), and calls it lazily behind a flag at each of the four sites that read the buffer: syslog, eventlog, stderr and the syslogger_setup_done path. With csvlog or jsonlog alone, the buffer is never built. Only src/backend/utils/error/elog.c changes.

**User-visible surface.**
- log_line_prefix '%l' now counts only messages for which the stderr format was built, per a reviewer's test; the author does not mention it.

**What the author claims.**
- With csvlog only and log_min_duration_statement = 0, a pgbench run of a 32MB statement went from about 3.41 to 3.80 tps.
- The patch avoids building the buffer "if possible"; the author claims no other effect.

**Thread state.** Chao Li liked the idea and suggested renaming build_stderr_message() and has_stderr_message, since syslog and eventlog also use the buffer. Diego tested five log_destination setups, found identical output, and raised the '%l' change and the repeated deferred-build check at four sites; his message is cut off and the author has not replied.

## 2. Prior attempts

### Skip vacuum log report code in lazy_scan_heap() if possible (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3404/](https://commitfest.postgresql.org/patch/3404/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAJcOf-fH5FGsF3mpPL1m5yHLeR0HdXWQp9RA=LnHu0Qd=eq9DA@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAJcOf-fH5FGsF3mpPL1m5yHLeR0HdXWQp9RA=LnHu0Qd=eq9DA@mail.gmail.com)
- **Date:** decided 2021-12-04; started 2021-10-29
- **Outcome:** Rejected (cf)
- **Who objected:** Andres Freund, Peter Geoghegan (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** lazy_scan_heap() runs once per relation, so the code path is not hot and the change saves no measurable time. Andres Freund also said the skip reduces test coverage of the report code, and Michael Paquier accepted that argument. Peter Geoghegan noted that his patch to unite VACUUM VERBOSE and autovacuum logging would remove the issue, so the entry was rejected in favour of that patch.
- **Reason (quote):** "I think this mostly reduces the coverage of the relevant code without any measurable speed gain." ([message](https://www.postgresql.org/message-id/20211203011344.lkxqyzhnm6jopubh%40alap3.anarazel.de))
- **What is different this time:** This patch skips formatting on a path that runs for every logged message, and message #1 shows a measured gain for csvlog-only logging. The moved code stays covered because the default stderr setup still runs it.


## 3. Objections a reviewer is likely to raise

### 1. Deferred build at four sites turns a structural invariant into a convention

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

A later patch adds a log destination, or a new branch in an existing one, that reads buf without repeating the flag check. With csvlog-only or jsonlog-only configurations, that branch formats uninitialized stack memory inside the error reporting path. Normal stderr setups and regression tests never hit it, so a user finds it as corrupt output or a crash.

Today initStringInfo(&buf) runs at the top of send_message_to_server_log(), so no destination can read an unbuilt buffer. The patch copies the check-and-build block to four sites, and nothing fails at compile time or in tests if a fifth site leaves it out (message #3). The author has not replied to this point. The defect would land in elog.c, the code that must work while other things fail.

Thread: [#3 (Diego, 2026-09-11)](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com)

**Question:** Can the build decision live in one place, such as an accessor that owns the flag or a predicate computed up front, so that a new consumer cannot skip it?

Counter-check: held. Message #3 raises this point and the thread has no answer. The patch is correct today, and the reviewer names two shapes that restore the invariant, so it stays fixable.

### 2. Log line number %l changes meaning without documentation

fixable · level 2, rubric item 10 · design · `standards-compat-1`

An administrator runs with log_destination = 'csvlog', then switches to 'stderr' with ALTER SYSTEM and pg_reload_conf(). After upgrade, '%l' in log_line_prefix restarts near 1 instead of continuing the per-session count. Log-correlation scripts that rely on that count see gaps or reuse they did not expect, and nothing in the release notes explains why.

Message #3 showed the effect: unpatched gives LINE=11, patched gives LINE=1. '%l' is documented as the "number of the log line for each session or process, starting at 1", and the patch makes it count only messages formatted for stderr, syslog or eventlog. The commit message claims no behaviour change, and the author has not answered the question in message #3. The new behaviour may be the better one, but a committer would likely want it chosen on purpose and documented, not left as a side effect.

Thread: [#3 (Diego, 2026-09-11)](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com)

**Question:** Is the new '%l' counting intended, and will the docs or the commit message say that it counts only lines rendered in the stderr format?

Counter-check: held. Message #3 shows LINE=11 unpatched and LINE=1 patched, and the author has not replied. The effect is narrow, since it needs a run-time change of log_destination, so fixable is the right severity.

### 3. Gain shown only for one extreme 32MB statement

fixable · level 1b, rubric item 4 · demand, maintenance burden ("who maintains this in 10 years"), performance on the hot path · `demand-1`

A committer pushes the patch on the strength of an 11% gain measured with a 32MB statement. Typical csvlog-only servers log statements of a few KiB and may see no measurable change. Meanwhile, everyone who later edits send_message_to_server_log() must remember the deferred-build check at each consumer of buf.

The only evidence is one 60-second, one-client pgbench run of a 32MB statement with log_min_duration_statement = 0 (message #1). No user report, field case or typical-workload number shows that csvlog-only or jsonlog-only servers lose real time to this buffer. The new rule that each consumer must build buf first applies to everyone who edits elog.c (message #3). For that trade to be worth it, the gain should show up beyond the worst case.

Thread: [#1 (cca5507, 2026-08-02)](https://www.postgresql.org/message-id/tencent_A00A556AC153E46588F0680E34ED4C43EF09%40qq.com), [#3 (Diego, 2026-09-11)](https://www.postgresql.org/message-id/6aa3ff5e.749efa0d.1234b.4ae7%40mx.google.com)

**Question:** What is the measured gain with csvlog or jsonlog alone on a realistic logging-heavy workload, for example many clients running ordinary-sized statements with log_min_duration_statement = 0?

Counter-check: held. The waste is real, and the default configuration pays one boolean test, per message #3. But the only number is one 60-second, one-client run of a 32MB statement, so the gain for ordinary statement sizes is not shown.

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
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each of the four consumers of buf in send_message_to_server_log() must now run the deferred-build check first. |
| performance on the hot path | **yes** | The patch changes send_message_to_server_log(), which runs for every logged message. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Can the build decision live in one place, such as an accessor that owns the flag or a predicate computed up front, so a new consumer cannot skip it?
2. Is the new '%l' counting intended, and will the docs or the commit message say that it counts only lines rendered in the stderr format?
3. What is the measured gain with csvlog or jsonlog alone when many clients run ordinary-sized statements with log_min_duration_statement = 0?
4. Has anyone run the patched code with log_destination = 'syslog' and compared the output with master?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows the error reporting code in elog.c and the syslogger paths would have to own it. None has spoken in the thread.
- **Liability:** The owner signs for correct behaviour of send_message_to_server_log() in every log_destination combination, including the untested syslog path, and for the '%l' change. The check-before-use rule lasts as long as the deferred build does.
- **Author follow-up:** The author has not replied to message #2 or message #3; no sign either way.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch touches only elog.c, and the shown gain is for csvlog-only logging of a 32MB statement. A reply to message #3 and a single-place build decision would make it a short review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Gain shown only for one 32MB statement, not for common logging loads. Refuted (not specific): Duplicate of demand-1. Both ask for numbers beyond the single 32MB run, and demand-1 states the trade more directly.

### Review angles

- prior-attempts: 0 objection(s). [P5] was rejected because its path ran once per relation, saved no measurable time and lost test coverage. That reason does not carry over here: send_message_to_server_log() runs for every logged message, message #1 shows a gain, and the default stderr setup still runs the moved code. The thread cites no earlier discussion of this idea. Whether the gain matters for typical workloads is a question for the performance and demand angles.
- standards-compat: 1 objection(s). From the standards and compatibility angle the patch adds no syntax, GUC, catalog object or dump state. Message #3 found identical log output in five configurations. The only user-visible change is '%l'.
- replication-ondisk: 0 objection(s). The patch changes only how send_message_to_server_log() in elog.c builds an in-memory buffer for server log output. It adds no on-disk format, WAL record, catalog change, wire protocol message or exported API, so nothing crosses a pg_upgrade, a standby or a client boundary. The '%l' counter change affects log file text only, and other angles cover it.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. It changes when elog.c formats the stderr log line, not who can set log_destination or what reaches each destination, and message #3 found identical output in five configurations. The risk of a future destination reading an uninitialized buffer is a maintainability concern for another angle.
- maintenance-fit: 2 objection(s). The patch touches only elog.c and adds one boolean test in the default stderr configuration (message #3), so hot-path cost for non-users is not a concern.
- already-exists: 0 objection(s). No existing feature, setting or extension stops send_message_to_server_log() from building the stderr-format buffer when only csvlog or jsonlog is in use. emit_log_hook could replace server logging entirely, but that is a different design, so this angle has nothing to raise. [P5] is about a cold code path, which another angle covers.
- demand: 1 objection(s). Message #3 confirms the premise, and the default stderr setup pays only one boolean test. The only demand concern is how large the gain is outside the extreme benchmark.

### Precedents retrieved

- P1: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant; reason summarised from its thread
- P2: [Issue a log message when the backtrace logged is cut off](https://commitfest.postgresql.org/patch/3547/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P3: [Fix log_line_prefix to display the transaction id (%x) for statements not in a transaction block](https://commitfest.postgresql.org/patch/4606/) (cf, Withdrawn, 2024-05-18) - judged not relevant; reason summarised from its thread
- P4: [Revert logical snapshot filename format change in SnapBuildSnapshotExists().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e9a31c0cc604b015363d0aa5b018e8cea9d1d9a7) (revert, Reverted, 2025-08-14) - judged not relevant
- P5: [Skip vacuum log report code in lazy_scan_heap() if possible](https://commitfest.postgresql.org/patch/3404/) (cf, Rejected, 2021-12-04) - used in the review; reason summarised from its thread
- P6: [Revert non-text output formats for pg_dumpall](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=7ca548f23a60887541493b2da47f2b2720c35ee3) (revert, Reverted, 2026-06-18) - judged not relevant
- P7: [Revert idea of zer-padding padding session id in log_line_prefix](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=8b06e6aba8773df8040b357a1cb058879ed2d94f) (revert, Reverted, 2013-05-06) - judged not relevant
- P8: [Fix slotsync worker busy loop causing repeated log messages](https://commitfest.postgresql.org/patch/6546/) (cf, Withdrawn, 2026-05-01) - judged not relevant; reason summarised from its thread
- P9: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P10: [SQL function to report log message](https://commitfest.postgresql.org/patch/314/) (cf, Returned with feedback, 2015-12-22) - judged not relevant; reason summarised from its thread
- P11: [Improving the heapgetpage function improves performance in common scenarios](https://commitfest.postgresql.org/patch/4524/) (cf, Withdrawn, 2023-09-06) - judged not relevant; reason summarised from its thread
- P12: [Improve pg_restore toc file parsing and format for better performances](https://commitfest.postgresql.org/patch/4509/) (cf, Returned with feedback, 2024-01-14) - judged not relevant; reason summarised from its thread
- P13: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant
- P14: [Avoid smgrimmedsync() during index build and add unbuffered IO API](https://commitfest.postgresql.org/patch/3508/) (cf, Returned with feedback, 2022-08-02) - judged not relevant
- P15: [log_destination=file](https://commitfest.postgresql.org/patch/1274/) (cf, Returned with feedback, 2018-03-02) - judged not relevant
- P16: [Force the old transactions logs cleanup even if checkpoint is skipped](https://commitfest.postgresql.org/patch/4619/) (cf, Returned with feedback, 2024-04-09) - judged not relevant
- P17: [skip empty transactions for logical replication](https://commitfest.postgresql.org/patch/2384/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P18: [Send numeric version to clients (server_version_num GUC_REPORT)](https://commitfest.postgresql.org/patch/752/) (cf, Rejected, 2016-08-30) - judged not relevant
- P19: [pg_upgrade data type check connection overhead reduction](https://commitfest.postgresql.org/patch/4200/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P20: [Improve the performance of nested loop join in the case of partitioned inner table](https://commitfest.postgresql.org/patch/4253/) (cf, Returned with feedback, 2023-08-01) - judged not relevant
- P21: [minimizing pg_stat_statements.track='none' performance overhead](https://commitfest.postgresql.org/patch/2092/) (cf, Withdrawn, 2019-04-15) - judged not relevant
- P22: [START/END line number for COPY FROM](https://commitfest.postgresql.org/patch/1940/) (cf, Rejected, 2019-01-30) - judged not relevant
- P23: [Iterating on IndexTuple attributes and nbtree page-level dynamic prefix truncation](https://commitfest.postgresql.org/patch/3078/) (cf, Returned with feedback, 2022-02-07) - judged not relevant
- P24: [Revert part of the previous patch that avoided using PLy_elog().](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=de479e2ed28925ddd2907cad3e2d5dddd3b97199) (revert, Reverted, 2012-07-05) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 99 s; cost reported by the CLI: $1.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 62 text field(s) checked, 6 finding(s), 6 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:20:20.983Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
