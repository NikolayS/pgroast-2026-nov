# Design pre-review: Print measurements on CTRL-C in pgbench

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7526](https://commitfest.postgresql.org/patch/7526/) (Needs review); authors: Roman Khapov (rkhapov)
- Thread: [Print pgbench results on SIGINT](https://www.postgresql.org/message-id/flat/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), started 2026-10-26
- Messages read: 3 of 3
- Patch prose read: v2-0001-pgbench-print-partial-results-on-SIGINT.patch, 0001-pgbench-print-partial-results-on-SIGINT.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is fatal, and the idea fits pgbench as a core tool. Three fixable points remain: already-exists-1 asks why `-T` with `-P` falls short, standards-compat-1 shows that exit(130) lets a calling shell loop continue, and maintenance-fit-2 notes that Windows behaviour is undefined. The author calls the patch a POC and leaves the exit code open in message #1, so these are open design points and do not count against the goal.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check how v2 handles in-flight transactions in the statistics, whether the handler is async-signal-safe, or how the one-second wait cap interacts with `-R` and `\sleep`.

## The proposal, restated

**Problem.** Developers run pgbench for a few seconds, press Ctrl-C, change code and run again. Today Ctrl-C kills pgbench and prints none of the statistics collected so far.

**Proposed design.** On the first SIGINT, pgbench stops starting new transactions, discards in-flight ones, prints the usual summary with the existing "Run was aborted" message, and exits with status 130. A second SIGINT uses the default handler and kills the process at once. All waits are capped at one second (MAX_IDLE_WAIT_USEC) so a thread idle in \sleep notices the signal.

**User-visible surface.**
- Ctrl-C (SIGINT) now prints the run summary instead of killing pgbench silently.
- Exit status 130 after a first SIGINT.
- Second Ctrl-C kills pgbench immediately.
- \sleep and other waits now wake at least once per second.

**What the author claims.**
- Useful for short manual runs that compare performance before and after a code change.
- Reviewer Kirill Reshke is +1 on the UX and on the double Ctrl-C behaviour.
- v2 fixes a race where workers could send new commands after the stop flag was set.

**Thread state.** Kirill Reshke reviewed v1, found a race in the stop-flag check, and pointed to the earlier thread "Make pgbench exit on SIGINT more reliably" for Windows handling. The author posted v2 with the fix; the exit code and Windows behaviour are still open.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-10-26; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Existing -T and -P options already cover short manual runs

fixable · level 1b, rubric item 3 · doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A developer who wants a 5-10 second comparison can run `pgbench -T 10 -P 1` and get the full summary plus per-second TPS and latency. The patch adds a new SIGINT path, a new exit status and a one-second wait cap that every pgbench run then carries. The open Windows question has to be resolved and maintained for a case these options already handle.

Message #1 describes runs of a known short length ("5-10 seconds"). That is the use case for `-T`, which ends with the normal summary, and `-P` prints TPS and latency while the run is going, so a user who presses Ctrl-C has already seen the numbers. The proposal does not mention either option or say why they fall short. Without that, the gain looks like convenience: no need to choose the duration in advance. A committer would likely ask whether that convenience is worth the permanent signal-handling code.

Thread: [#1 (Roman Khapov, 2026-10-26)](https://www.postgresql.org/message-id/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), [#2 (Kirill Reshke, 2026-10-27)](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com)

**Question:** What does a Ctrl-C summary give that `-T` with `-P 1` does not, for the short comparison runs described in message #1?

Counter-check: held. `-T 10 -P 1` gives the numbers for the 5-10 second runs in message #1, and the thread does not say why that falls short. One +1 in message #2 does not show the need is common.

### 2. Exit 130 instead of re-raising SIGINT breaks shell loop interruption

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-1`

A user runs pgbench inside a bash loop or a Makefile, for example for each scale factor, and presses Ctrl-C. Bash sees pgbench exit normally with status 130 rather than die from SIGINT, so it keeps going and starts the next pgbench run. The user has to press Ctrl-C again for every remaining iteration.

Today pgbench dies from SIGINT, and calling shells rely on that to stop the whole script. The usual convention for programs that catch SIGINT is to clean up, reset the handler and re-raise the signal, not to exit(130). The author says in message #1 that the exit code is still open. The patch adds no documentation hunk, so the documented exit statuses, 0, 1 and 2 with partial results, would gain an undocumented fourth value.

Thread: [#1 (Roman Khapov, 2026-10-26)](https://www.postgresql.org/message-id/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), [#2 (Kirill Reshke, 2026-10-27)](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com), [#3 (Roman Khapov, 2026-10-28)](https://www.postgresql.org/message-id/39384C28-7D89-4E8A-A85F-952309075357%40yandex-team.ru)

**Question:** After printing the summary, will pgbench re-raise SIGINT so calling shells see a signal death, and how will the exit-status documentation change?

Counter-check: held. Bash continues a loop when the child catches SIGINT and exits normally, even with status 130, so the scenario is real. The author leaves the exit code open in message #1, and the patch has no documentation hunk for the new status.

### 3. Windows behaviour left undefined for a cross-platform tool

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-2`

A Windows user presses Ctrl-C during a run. pgbench may print nothing, print a summary while threads keep running, or hang until a wait expires. The user finds out only on that platform, and the bug report then goes to a committer who has no Windows setup.

Windows delivers console Ctrl-C through a separate handler thread rather than POSIX SIGINT, so a design built on signal state and EINTR does not carry over unchanged. Message #2 points to an earlier thread on exactly this issue, and message #3 only promises to read it. The touched files list only pgbench.c, with no tests and no documentation for either platform.

Thread: [#2 (Kirill Reshke, 2026-10-27)](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com), [#3 (Roman Khapov, 2026-10-28)](https://www.postgresql.org/message-id/39384C28-7D89-4E8A-A85F-952309075357%40yandex-team.ru)

**Question:** What does a first and a second Ctrl-C do on Windows with v2, and how does a test show that it matches the Unix behaviour?

Counter-check: held. Windows runs the SIGINT handler in a separate thread and has no EINTR, and the thread does not say what v2 does there. Message #3 only promises to read the earlier thread, so the question is open.

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
| backpatchability | **yes** | see objection standards-compat-1: Exit 130 instead of re-raising SIGINT breaks shell loop interruption |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a SIGINT path, a new exit status and a one-second wait cap in pgbench.c, with Windows handling still open. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | `pgbench -T 10 -P 1` already gives a summary and per-second numbers for the short runs in message #1. |

## 5. Questions the author must answer before anyone reviews code

1. What does a Ctrl-C summary give that `-T` with `-P 1` does not, for the short comparison runs described in message #1?
2. After printing the summary, will pgbench reset the handler and re-raise SIGINT so a calling shell loop or Makefile stops, or exit with status 130?
3. What do a first and a second Ctrl-C do on Windows with v2?
4. How do the discarded in-flight transactions affect the TPS, latency and per-script numbers in the summary?
5. How will the pgbench documentation on exit status and interrupted runs change?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Someone who knows pgbench's thread and wait loop and frontend signal handling, including Windows console Ctrl-C handling.
- **Liability:** The owner signs for the SIGINT path in pgbench on every platform, the exit-status contract that scripts will depend on, and the one-second wait cap, for as long as pgbench ships.
- **Author follow-up:** The author posted v2 two days after the review in message #2 and said he will read the earlier Windows thread.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** low
- **Why:** The patch is about 4.6 KB in one file and serves developers who run pgbench by hand, though the thread shows only one such workflow. The exit behaviour and the Windows behaviour need an answer before code review is useful.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Stop-flag checks spread through the client state machine. Refuted (already handled): Message #3 says v2 checks the flag before the call to advanceConnectionState(), so a state added inside that function is already behind the check. The remaining request for a TAP test is below level 2.
- `demand-1` Demand rests on one workflow that -T and -P already cover. Refuted (not specific): Duplicate of already-exists-1, which makes the same `-T` and `-P` argument with a more concrete question for the author.

### Review angles

- prior-attempts: 0 objection(s). No precedent found; the corpus search returned no earlier attempt at the same or a similar idea. Message #2 points to an earlier thread, "Make pgbench exit on SIGINT more reliably", but only as possible context on Windows handling, with no outcome or rejection reason, so no earlier objection can be checked against this proposal.
- standards-compat: 1 objection(s). Beyond the exit-status contract, nothing else in this angle seems to need an objection. The proposal adds no syntax, GUC, catalog object or dump format. It reuses the existing 'Run was aborted' message, so scripts can tell partial output apart from a normal result.
- replication-ondisk: 0 objection(s). The patch changes only client-side signal handling in src/bin/pgbench/pgbench.c. It adds no on-disk format, WAL record, catalog entry, wire protocol message or server ABI change. Discarded in-flight transactions close as an ordinary client disconnect, so the server rolls them back, which is existing behaviour. This angle has nothing to object to.
- security: 0 objection(s). The proposal changes only how the pgbench client handles SIGINT in its own process. It grants no privilege, exposes no data, and touches no server code, authentication or connection setup. Any change in server-side work after Ctrl-C stays within what the same user could already do with the same connection.
- maintenance-fit: 2 objection(s). From the maintenance angle the idea fits core: pgbench is a core tool, the change touches one file and costs nothing on the server. The one-second wait cap adds a negligible wakeup cost, so it is not raised.
- already-exists: 1 objection(s). Only one objection from this angle. No extension or external tool replaces the feature, and no precedent applies.
- demand: 1 objection(s). The likely user group is wider than one person, since many people press Ctrl-C on pgbench. The thread does not show this, so the objection asks for evidence and does not argue against the idea.

### Precedents retrieved

- P1: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant; reason summarised from its thread
- P2: [pgbench --continue-on-error: clarify TPS and failure reporting](https://commitfest.postgresql.org/patch/6942/) (cf, Withdrawn, 2026-06-28) - judged not relevant; reason summarised from its thread
- P3: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P4: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant; reason summarised from its thread
- P5: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P6: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - judged not relevant; reason summarised from its thread
- P7: [Revert "Add some temporary code to record stack usage at server process exit."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=96112ee7c60557bb192a9aa07b514db2400fd45e) (revert, Reverted, 2016-07-10) - judged not relevant
- P8: [pgbench allow '=' in \set](https://commitfest.postgresql.org/patch/246/) (cf, Rejected, 2015-05-13) - judged not relevant; reason summarised from its thread
- P9: [merging pgbench logs](https://commitfest.postgresql.org/patch/193/) (cf, Returned with feedback, 2015-10-03) - judged not relevant; reason summarised from its thread
- P10: [Skip collecting decoded changes of already-aborted transactions](https://commitfest.postgresql.org/patch/4436/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P11: [add assertion for palloc in signal handlers](https://commitfest.postgresql.org/patch/6498/) (cf, Withdrawn, 2026-08-24) - judged not relevant
- P12: [Custom signals and handlers for extension](https://commitfest.postgresql.org/patch/1430/) (cf, Returned with feedback, 2020-11-26) - judged not relevant
- P13: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant
- P14: [Revert "Track statistics for spilling of changes from ReorderBuffer".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d973747281caece520236e93d255c654cc613ec9) (revert, Reverted, 2020-07-13) - judged not relevant
- P15: [Summary Sort workers Stats in EXPLAIN ANALYZE](https://commitfest.postgresql.org/patch/3601/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P16: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant
- P17: [Partial aggregates push down](https://commitfest.postgresql.org/patch/3369/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P18: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P19: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4019/) (cf, Returned with feedback, 2024-01-27) - judged not relevant
- P20: [Partial sort](https://commitfest.postgresql.org/patch/409/) (cf, Returned with feedback, 2017-01-31) - judged not relevant
- P21: [Revert refactoring of restore command code to shell_restore.c](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2f6e15ac93c58c1140e4a4affe61e78f7346497a) (revert, Reverted, 2023-02-05) - judged not relevant
- P22: [Partial aggregates push down](https://commitfest.postgresql.org/patch/4857/) (cf, Returned with feedback, 2024-10-08) - judged not relevant
- P23: [Revert "Get rid of the dedicated latch for signaling the startup process".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=00f690a239932e477f25120d19b08aacdc30deb7) (revert, Reverted, 2020-12-17) - judged not relevant
- P24: [pg_buffercache: Add per-relation summary stats](https://commitfest.postgresql.org/patch/6555/) (cf, Returned with feedback, 2026-05-04) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (already-exists-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 114 s; cost reported by the CLI: $1.03
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 5 from cache, 3 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 70 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T04:04:28.576Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
