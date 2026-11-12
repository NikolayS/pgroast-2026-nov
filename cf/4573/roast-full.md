# Design pre-review: pg_ctl start may return 0 even if the postmaster has been already started on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 4573](https://commitfest.postgresql.org/patch/4573/) (Waiting on Author); authors: Kyotaro Horiguchi (horiguti)
- Thread: [pg_ctl start may return 0 even if the postmaster has been already started on Windows](https://www.postgresql.org/message-id/flat/TYAPR01MB586654E2D74B838021BE77CAF5EEA%40TYAPR01MB5866.jpnprd01.prod.outlook.com), started 2023-09-07
- Messages read: 43 of 43
- Patch prose read: v3-0001-Disable-autoruns-for-cmd.exe-on-Windows.patch, v3-0002-Improve-pg_ctl-postmaster-process-check-on-Window.patch, v3-0003-Remove-short-sleep-from-001_start_stop.pl.patch, move_checking.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: the bug is stated in message #1 and committers agree it should be fixed. The surviving objections are fixable level 2 points about the CreateProcess() design. prior-attempts-1 is the strongest, because message #34 reverted /D for the same user-visible effect. maintenance-fit-1 needs a committer who knows Windows token handling to sign.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can tell whether v6 shares token code with CreateRestrictedProcess(), keeps the 2-second wait, or cleans up the two global PID variables. The Unicode executable check failures that Harrison reports are also unexplained.

## The proposal, restated

**Problem.** On Windows, `pg_ctl start` can return 0 when the cluster was already started less than 2 seconds earlier. pg_ctl launches the postmaster through cmd.exe, so it knows only the shell PID and cannot check that postmaster.pid belongs to its own child. Scripts and tools such as pg_upgrade that rely on the exit code get a wrong result.

**Proposed design.** The original patch keeps cmd.exe, adds /D to disable autorun, and walks the process table for a postgres.exe child of the launcher shell to get the real postmaster PID. Later messages offer two other designs: pg_ctl passes a random token that the postmaster writes to postmaster.pid, or pg_ctl starts postgres.exe directly with CreateProcess() and handle-based redirection. The latest patches follow the CreateProcess() design.

**User-visible surface.**
- `pg_ctl start` on Windows returns failure when another postmaster already runs in the data directory.
- `pg_ctl start` on Windows may print different error messages, consolidated to "could not start server".
- The launcher cmd.exe runs with /D, so registry AutoRun commands no longer run (committed, then reverted for v17).
- The token design would add a new value in postmaster.pid and a way for pg_ctl to pass it to the postmaster.

**What the author claims.**
- Process table walking fixes the bug, and several reviewers report it passes CI.
- Removing the short sleep in 001_start_stop.pl reproduces the bug, and the patch fixes it.
- The CreateProcess() patch passes all Windows regression tests, including CI with elevated privileges.
- CreateProcess() fixes the root cause with about 250 lines of Windows-only code and keeps privilege dropping.
- The latest refinement passes LocalSystem, LocalService and elevated tests on x64 and ARM64 with paths that contain spaces and U+00E9.

**Thread state.** Committers have not chosen among process table walking, the token, and CreateProcess(); Álvaro Herrera favors CreateProcess() for master and a smaller fix for back branches. The entry has moved through 13 CommitFests, and a new v6 patch from October 2026 has no reply.

## 2. Prior attempts

No precedent found (corpus searched for items before 2023-09-07; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Dropping cmd.exe brings back the AutoRun change reverted for v17

fixable · level 2, rubric item 12 · prior rejection, backpatchability, maintenance burden ("who maintains this in 10 years") · `prior-attempts-1`

A Windows administrator sets a cmd.exe AutoRun entry, for example `chcp` or environment variables, that the postmaster inherits today through the launcher shell. After the CreateProcess() change, `pg_ctl start` runs postgres.exe directly, so the entry never runs. The administrator finds out only when server log encoding or environment differs after an upgrade.

Message #32 argued that it is not Postgres's job to second-guess the user's AutoRun settings, and that /D on only some %comspec% callers is inconsistent. Message #34 reverted /D for that reason. Running postgres.exe without cmd.exe has the same user-visible effect for pg_ctl start, while pgwin32_system() keeps running AutoRun. Messages #40-#43 do not discuss this.

Thread: [#11 (Kyotaro Horiguchi, 2023-09-22)](https://www.postgresql.org/message-id/20230922.161551.320043332510268554.horikyota.ntt%40gmail.com), [#32 (Noah Misch, 2024-06-30)](https://www.postgresql.org/message-id/20240630021211.f3.nmisch%40google.com), [#33 (Michael Paquier, 2024-07-05)](https://www.postgresql.org/message-id/ZoeCemTDhF0OxuB2%40paquier.xyz), [#34 (Michael Paquier, 2024-07-08)](https://www.postgresql.org/message-id/Zos5ojel68kBT79X%40paquier.xyz), [#40 (Bryan Green, 2025-10-24)](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#42 (Álvaro Herrera, 2026-02-02)](https://www.postgresql.org/message-id/202602021323.y4ctu6t5biso%40alvherre.pgsql)

**Question:** Why is skipping AutoRun acceptable for pg_ctl start under CreateProcess() when the same effect from /D was reverted, and what happens to other %comspec% callers?

Counter-check: held. Message #32 objected to skipping AutoRun in only some callers, and message #34 reverted /D on that basis. Direct launch has the same effect for pg_ctl start, and messages #40-#43 do not address it.

### 2. Second copy of the privilege-dropping logic for the postmaster launch

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), security / privileges · `maintenance-fit-1`

A later patch changes which groups or privileges CreateRestrictedProcess() removes, for example after a Windows change. The new CreateProcessAsUser() path in start_postmaster() keeps the old token rules. An elevated pg_ctl then starts a postmaster with more rights than intended, and nothing in CI reports the difference.

Message #40 says the new code builds its own restricted token, "matching the behavior of the existing CreateRestrictedProcess() function". Matching by copy means two places must stay in sync, and only elevated-privilege tests would show a gap. Message #43 adds a refinement that opens log files under the restricted identity, so this security-sensitive code grows further. The prose does not say whether v6 shares one helper with CreateRestrictedProcess().

Thread: [#40 (Bryan Green, 2025-10-24)](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#43 (harrisontbooth(at)gmail(dot)com, 2026-10-17)](https://www.postgresql.org/message-id/CAAH-eSX%3DM5fsShzL401VB7t82qiAsv2difOHnPd0qWA%2BHBMc-w%40mail.gmail.com)

**Question:** Does the direct launch reuse the same token-building code as CreateRestrictedProcess(), so one change covers both paths?

Counter-check: held. Message #40 says the new path builds a restricted token that matches CreateRestrictedProcess(). Neither the thread nor the patch prose says the two paths share code, so the risk that they drift apart stands.

### 3. pg_ctl -o and -l lose cmd.exe parsing without a stated contract

fixable · level 2, rubric item 12 · design, backpatchability · `standards-compat-2`

A Windows service is registered with pg_ctl register -o containing %PGDATA2% or caret-escaped quotes, and the stored command line is expanded by cmd.exe at each start. After the change, CreateProcess() passes the string literally and the postmaster rejects the option or reads the wrong path. The user finds out at the next service restart.

Today start_postmaster() builds a cmd.exe command line, so -o text and the -l redirection go through cmd.exe quoting, %VAR% expansion and append semantics. Message #40 describes handle-based redirection, but not whether -o text is parsed the same way or whether the log file is still appended to. Message #43 tests paths with spaces and U+00E9, but not shell metacharacters or variables in -o.

Thread: [#40 (Bryan Green, 2025-10-24)](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#43 (harrisontbooth(at)gmail(dot)com, 2026-10-17)](https://www.postgresql.org/message-id/CAAH-eSX%3DM5fsShzL401VB7t82qiAsv2difOHnPd0qWA%2BHBMc-w%40mail.gmail.com)

**Question:** Is the postmaster command line, including -o text with %VAR%, ^ and quotes, and the -l append behaviour, identical with and without cmd.exe, and is any difference documented?

Counter-check: held. pg_ctl start passes -o text and the `>>` log redirection through cmd.exe today, and messages #40 and #43 do not say that parsing and append behaviour stay the same. The service scenario is weak support: as far as I know, service start already launches postgres.exe without cmd.exe.

### 4. Fixed 2-second wait to detect postgres -C

fixable · level 2, rubric item 9 · design, maintenance burden ("who maintains this in 10 years") · `maintenance-fit-3`

On a loaded Windows CI machine, a postgres -C child or an early-failing postmaster takes longer than 2 seconds to exit. pg_ctl takes the wrong branch and the test fails only sometimes. Developers then find intermittent failures that look like unrelated Windows noise.

Message #40 says the direct launch waits 2 seconds when there is no log file, to tell postgres -C apart from server startup. That puts a new timing threshold into code whose main fault is already a 2-second clock allowance (message #1). Timing thresholds tend to need retuning, and the material does not say whether v6 keeps this wait or replaces it with an exit-status check.

Thread: [#1 ("Hayato Kuroda (Fujitsu)", 2023-09-07)](https://www.postgresql.org/message-id/TYAPR01MB586654E2D74B838021BE77CAF5EEA%40TYAPR01MB5866.jpnprd01.prod.outlook.com), [#40 (Bryan Green, 2025-10-24)](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com)

**Question:** Can the launch tell postgres -C from server startup without a fixed delay, and does v6 still contain the 2-second wait?

Counter-check: held. Message #40 states a fixed 2-second wait to tell `postgres -C` from server startup, and no later message says v6 removes it. A slow exit on a loaded machine would take the wrong branch.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The token design would add a value to postmaster.pid, and pg_upgrade relies on the pg_ctl start exit code. |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The CreateProcess() design builds its own restricted token to drop privileges for the postmaster. |
| backpatchability | **yes** | It is a bug fix, the /D change was reverted for v17, and message #42 suggests a smaller fix for back branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | About 250 lines of Windows-only launch and token code in pg_ctl.c that few reviewers can check. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is skipping cmd.exe AutoRun acceptable under CreateProcess() when message #34 reverted /D for the same effect, and what happens to other %comspec% callers?
2. Does the direct launch share the token-building code with CreateRestrictedProcess(), so one change covers both paths?
3. Are -o text with %VAR%, ^ and quotes, and the -l append behaviour, the same with and without cmd.exe?
4. Does v6 still wait 2 seconds to detect postgres -C, and can the launch tell the two cases apart without a fixed delay?
5. Which branches does the fix target, and does the bug exist in v16 as message #36 questions?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Álvaro Herrera states a design preference in message #42; the owner needs to know Windows process creation and restricted tokens, and message #38 notes that reviewers are not Windows experts.
- **Liability:** The owner signs for Windows-only launch and privilege-dropping code in pg_ctl for as long as Windows is supported, and for any behaviour change seen by AutoRun or -o users.
- **Author follow-up:** A v6 patch arrived in October 2026 and has no reply; the entry is Waiting on Author, and the material does not say who maintains the current patch.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The bug affects Windows scripts and tools that trust the pg_ctl start exit code, with CI flakiness as the only shown impact. The entry has moved through 13 CommitFests without a design decision, and v6 needs a committer who knows Windows to review it.

## Appendix

### Refuted in the counter-check

- `standards-compat-1` Direct CreateProcess() launch drops cmd.exe AutoRun, the change reverted for v17. Refuted (not specific): Duplicate of prior-attempts-1, which states the same AutoRun point and also covers the inconsistency with pgwin32_system().
- `security-1` Direct launch adds a second privilege-dropping path for the postmaster. Refuted (not specific): Duplicate of maintenance-fit-1. Its own scenario depends on an unshown difference between the two token paths, and message #43 reports passing LocalSystem, LocalService and elevated tests.
- `maintenance-fit-2` Different Windows launch mechanisms on master and back branches. Refuted (no failure scenario): Message #42 only suggests the split with "perhaps"; no patch proposes it. A smaller fix in back branches than on master is existing Postgres practice, and the scenario describes a normal backpatching cost.

### Review angles

- prior-attempts: 1 objection(s). No listed precedent matches. [P1] adds a postmaster.pid line, but its timestamp concern does not apply to a random token; [P5] and [P23] concern other pg_ctl behaviour. The only prior rejection found is inside this thread: /D was committed, then reverted (messages #21-#34).
- standards-compat: 2 objection(s). Two objections from the standards and compatibility angle. Reporting failure when a server already runs fixes a bug and is not a compatibility objection.
- replication-ondisk: 0 objection(s). The CreateProcess() design changes only how pg_ctl launches postgres.exe on Windows; it adds no WAL, catalog, on-disk, protocol or extension API state. Only the token design, which the latest patches do not follow, would add a postmaster.pid line, so this angle has nothing to object to.
- security: 1 objection(s). The bug itself has no privilege aspect. Only the CreateProcess() design touches security, through its own token restriction.
- maintenance-fit: 3 objection(s). The goal is a real bug fix that a committer supports in principle (messages #2, #42), so no objection is raised at levels 1a or 1b.
- already-exists: 0 objection(s). No existing feature, option or extension gives a correct `pg_ctl start` exit code on Windows, and checking `pg_ctl status` first is still racy. A possible overlap between the CreateProcess() launcher and the existing CreateRestrictedProcess() cannot be judged without the code.
- demand: 0 objection(s). This is a correctness bug, not a feature: `pg_ctl start` on Windows can return 0 when it did not start the server. Michael Paquier (message #2), Robert Haas (message #38) and Álvaro Herrera (message #42) agree it should be fixed, so there is no demand case to make against it. The only evidence of impact is CI flakiness in pg_upgrade tests (message #1), with no field reports. That bears on how large a rewrite is justified, which is a design question for other angles.

### Precedents retrieved

- P1: [Let pg_ctl check the result of a postmaster config reload](https://commitfest.postgresql.org/patch/185/) (cf, Returned with feedback, 2015-08-25) - used in the review; reason summarised from its thread
- P2: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P3: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P4: [Revert "pg_upgrade: Fix quoting of some arguments in pg_ctl command"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=4e818866e57aa9ec13821fd36a2ff60e00e6dea6) (revert, Reverted, 2020-02-10) - judged not relevant
- P5: [Revert "pg_ctl: Add idempotent option"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=187ca5e8e90baacc2d36c8bd0b08040f33c07fa1) (revert, Reverted, 2013-04-30) - used in the review
- P6: [Revert "Temporarily make pg_ctl and server shutdown a whole lot chattier."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c5e9b771275b93b09eec6b760677fe6c5e726ab2) (revert, Reverted, 2016-02-10) - judged not relevant
- P7: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P8: [Improve logrotate test so that it meaningfully exercises syslogger.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d5c22d28f8c161aa729d841aeec936c646044d4) (revert, Partially reverted, 2019-05-19) - judged not relevant
- P9: [Revert commits 290acac92b and 8a7e9e9dad.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d) (revert, Reverted, 2019-11-30) - judged not relevant
- P10: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (revert, Reverted, 2017-04-24) - judged not relevant
- P11: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - judged not relevant; reason summarised from its thread
- P12: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant; reason summarised from its thread
- P13: [Revert "Remove reset of testtablespace from pg_regress on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=61be85afabba0cd85bb1bcaacbea8efa8641f564) (revert, Reverted, 2020-07-10) - judged not relevant
- P14: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P15: [Teach autovacuum.c to launch workers to advance table age without attendant antiwraparound cancellation behavior](https://commitfest.postgresql.org/patch/4027/) (cf, Withdrawn, 2023-01-26) - judged not relevant; reason summarised from its thread
- P16: [Changing the autovacuum launcher scheduling; oldest table first algorithm](https://commitfest.postgresql.org/patch/1573/) (cf, Returned with feedback, 2018-11-30) - judged not relevant; reason summarised from its thread
- P17: [reduce impact of lengthy startup and checkpoint tasks](https://commitfest.postgresql.org/patch/3448/) (cf, Returned with feedback, 2023-09-01) - judged not relevant; reason summarised from its thread
- P18: [ALTER TABLE - check for column existence before starting operations](https://commitfest.postgresql.org/patch/1621/) (cf, Returned with feedback, 2018-10-02) - judged not relevant; reason summarised from its thread
- P19: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant
- P20: [Remove pgpid_t type, use pid_t instead](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5035c93c8a5ac6804da79c67403460348b381924) (revert, Partially reverted, 2022-10-22) - judged not relevant
- P21: [Support multiple password-based authentication tokens concurrently](https://commitfest.postgresql.org/patch/3748/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P22: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant
- P23: [Windows Service status is set to "SERVICE_RUNNING" when timeout is occurred (version 9.6 or earlier).](https://commitfest.postgresql.org/patch/1970/) (cf, Returned with feedback, 2019-11-25) - used in the review
- P24: [tablesync early exit](https://commitfest.postgresql.org/patch/3062/) (cf, Withdrawn, 2022-04-05) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-fable-5-2, synthesis claude-fable-5-2; no tools given to any of them
- LLM calls: 18; wall time: 146 s; cost reported by the CLI: $3.45
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 82 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T00:54:20.555Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
