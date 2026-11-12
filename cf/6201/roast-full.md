# Design pre-review: Fix orphaned backend processes on Windows using Job Objects

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6201](https://commitfest.postgresql.org/patch/6201/) (Needs review); authors: Bryan Green (lispyone)
- Thread: [\[PATCH\] Fix orphaned backend processes on Windows using Job Objects](https://www.postgresql.org/message-id/flat/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), started 2025-11-03
- Messages read: 10 of 10
- Patch prose read: v2-0001-Use-Windows-Job-Objects-to-prevent-orphaned-child.patch, 0001-Use-Windows-Job-Objects-to-prevent-orphaned-child-pr.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection is against the idea, and two committers in the thread support terminating children from the kernel. Three fixable points remain. The commit message and docs give a cause that message #7 withdrew (demand-1). Job membership also covers helper programs that are not backends (standards-compat-1). The job's name and access rights are not stated (security-1). Confidence is medium because security-1 rests on an inference from message #10.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, the job's name, security descriptor, handle inheritance and breakaway flags could not be checked. The interaction with a job that pg_ctl creates in service mode is also unchecked.

## The proposal, restated

**Problem.** On Windows, backends keep running after the postmaster crashes or is killed with taskkill /F, so they hold locks and shared memory and block a clean restart until someone kills them by hand. Backends created suspended, or stuck in a loop or holding a spinlock, never see the postmaster death event. This affects anyone who runs Postgres on Windows.

**Proposed design.** PostmasterMain() creates a Job Object with JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE and assigns the postmaster to it, so all children inherit membership. When the postmaster exits for any reason, the job handle closes and the kernel terminates all children. If the process is already in a job and assignment fails, the server logs a message and continues without orphan protection.

**User-visible surface.**
- New documentation section "Process Management on Windows" in runtime.sgml.
- New log message when Job Object creation or assignment fails.
- Backends on Windows now end at once when the postmaster dies, where before they kept running.

**What the author claims.**
- The kernel enforces the cleanup with no polling and no race conditions, unlike inherited event handles or shared memory flags.
- Clean shutdown is unaffected, and a crash of a child does not terminate the postmaster.
- Tested on Windows 10/11 with MSVC and MinGW; backends end about 100-200ms after the postmaster is killed.
- It also covers children killed while still suspended, which the existing event handle cannot reach.
- The code is about 200 lines and falls back to current behaviour on any failure.

**Thread state.** Andres Freund asked for a test, and v2 added one to 013_crash_restart.pl; Thomas Munro gave a general +1 and plans to build his subprocess work on it. Harrison Booth posted a rebase plus an unnamed-job patch and asked the author to fix a comment and the access-denied message, with no reply visible.

## 2. Prior attempts

### Use SIGIO to detect postmaster death (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3066/](https://commitfest.postgresql.org/patch/3066/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CA+hUKGK1607VmtrDUHQXrsooU=ap4g4R2yaoByWOOA3m8xevUQ@mail.gmail.com](https://www.postgresql.org/message-id/flat/CA+hUKGK1607VmtrDUHQXrsooU=ap4g4R2yaoByWOOA3m8xevUQ@mail.gmail.com)
- **Date:** decided 2021-07-13; started 2020-09-17
- **Outcome:** Withdrawn (cf)
- **Who objected:** Thomas Munro (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author withdrew the patch after finding that only the last backend to call F_SETOWN on the shared pipe receives the signal. A fix would need a separate pipe per backend, which he judged unwarranted at that point. Heikki Linnakangas and Tom Lane had reviewed it without objection.
- **Reason (quote):** "On closer inspection, only the last backend to start up and do F_SETOWN on the pipe receives the signal." ([message](https://www.postgresql.org/message-id/CA%2BhUKGLua5Pnx%2Brjjxg%3D3OPO4oOjtUPjeUYeZXh7TkH3gA7-Uw%40mail.gmail.com))
- **What is different this time:** P1 signalled backends on Unix through SIGIO on a shared pipe, and only one backend got the signal. This proposal targets Windows and has the kernel terminate every job member, so that failure does not arise.


## 3. Objections a reviewer is likely to raise

### 1. Stated problem and the reproduction in the commit message do not match

fixable · level 1a, rubric item 1 · problem statement · `demand-1`

A committer reads the commit message and docs, which say the taskkill /F case orphans backends because existing mechanisms need postmaster code to run at exit. Thomas Munro says PostmasterHandle is signalled by the kernel, and the author agreed. The commit would then record a wrong cause, and later readers cannot tell which cases the patch fixes and which the handle-inheritance patches fix.

Message #7 says the wording of the main reproduction in message #1 was imprecise. The likely real cause is inherited handles, which other CF entries address. The cases left to this patch are children still suspended, infinite loops and spinlock holds (messages #7 and #9). Thomas notes in message #8 that he recalls no recent "manual orphan cleanup needed" reports, so the field-visible problem may be smaller than the commit message says. He also agrees that preemptive termination is the right design.

Thread: [#1 (Bryan Green, 2025-11-03)](https://www.postgresql.org/message-id/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), [#6 (Thomas Munro, 2025-11-06)](https://www.postgresql.org/message-id/CA%2BhUKGLVBGE2KkzLaDkKX9t7%3Dt2BvjtOLXef5NnMv4cAZyoz7w%40mail.gmail.com), [#7 (Bryan Green, 2025-11-06)](https://www.postgresql.org/message-id/f60065bd-bed3-4e78-beb6-7e81d5e3875d%40gmail.com), [#8 (Thomas Munro, 2025-11-07)](https://www.postgresql.org/message-id/CA%2BhUKGJo6hu6GToiXarBRF%2BAqhFPnzMTW2Nksm0x-%2B9m2%3DdskQ%40mail.gmail.com), [#9 (Bryan Green, 2025-11-19)](https://www.postgresql.org/message-id/5e80ef94-fa47-4110-9e3a-7007a4ecef14%40gmail.com)

**Question:** Which failure cases remain once the handle-inheritance patches land, and will the commit message and docs be rewritten to name only those cases?

Counter-check: held. Message #7 concedes the wording about existing mechanisms was loose, yet the v2 commit message still says they are less reliable for that reason. The commit message and docs do not name the cases the patch fixes: suspended children, loops, spinlock holds.

### 2. Job membership kills detached helper programs, even at clean shutdown

fixable · level 2, rubric item 13 · design · `standards-compat-1`

A Windows user's archive_command or COPY PROGRAM starts a detached helper, for example with `start /b`, that should finish an upload after the command returns. After upgrade the helper is still a job member. When the postmaster exits, even cleanly, the kernel kills it mid-upload, and the user finds out from missing or partial files.

The postmaster is assigned to the job, so every descendant inherits membership, not only backends, and KILL_ON_JOB_CLOSE fires on any postmaster exit. Message #1 says clean shutdown leaves "nothing left to kill", but that holds only for processes the postmaster waits for. Today such grandchildren survive the postmaster on Windows, so this changes behaviour for existing scripts and extensions without any request from the user. Neither the documentation hunk nor the thread covers archive_command, restore_command, COPY PROGRAM or extension-spawned processes, and the brief lists this as unclear.

Thread: [#1 (Bryan Green, 2025-11-03)](https://www.postgresql.org/message-id/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), [#7 (Bryan Green, 2025-11-06)](https://www.postgresql.org/message-id/f60065bd-bed3-4e78-beb6-7e81d5e3875d%40gmail.com)

**Question:** Which processes outside the postmaster's own children stay in the job, and will the job allow breakaway or will the documentation state that they are killed on any postmaster exit?

Counter-check: held. Job membership passes to every descendant unless breakaway is allowed, and the kernel kills members when the last handle closes, also at clean shutdown. The thread and the documentation hunk do not cover archive_command, COPY PROGRAM or extension-started helpers.

### 3. Named job object can be pre-created or opened by another local process

fixable · level 2, rubric item 7 · security / privileges · `security-1`

A local account that is not the service account creates a job object with the expected name before the postmaster starts. CreateJobObject then returns a handle to that existing object, and the other account keeps its own handle. It can call TerminateJobObject to kill every backend, or hold the job open so orphans survive. The DBA sees only unexplained crashes or orphaned backends.

Message #10 adds a separate patch to use an unnamed job, which suggests v2 names its job object. Message #10 also says that naming the job, or leaving it unnamed, does not set the access policy. The patch prose does not give the name, the namespace, the security descriptor, or whether ERROR_ALREADY_EXISTS counts as a failure. A Windows named kernel object with a predictable name and default rights is a known way to take over another process's object, and the author has not answered message #10.

Thread: [#10 (Harrison Booth, 2026-10-16)](https://www.postgresql.org/message-id/CAAH-eSVh63%2B_OwacE8dkhc%2BVZkAwLcScumA-ChrReeodFmDuFw%40mail.gmail.com)

**Question:** Does the postmaster create an unnamed job with a restrictive security descriptor and a non-inheritable handle, and does it refuse a job object that already exists?

Counter-check: held. The job name is an inference from message #10, but that message adds an unnamed-job patch and its access question has no reply. The patch prose does not give the name, the security descriptor, or how an existing object is handled.

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
| security / privileges | **yes** | The postmaster creates a kernel job object whose name, access rights and handle inheritance the patch prose does not state. |
| backpatchability | **yes** | The entry is tagged Bugfix but changes process behaviour on Windows; the author targets a major release only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The patch adds a Windows-only file, pg_job_object.c, that depends on job semantics under service managers and nested jobs. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which failure cases remain for this patch once the handle-inheritance patches land, and will the commit message and docs name only those cases?
2. Are helper programs started from archive_command, restore_command, COPY PROGRAM or an extension killed when the postmaster exits cleanly, or does the job allow breakaway?
3. Is the job object named, what security descriptor does it get, and does the postmaster refuse a job object that already exists?
4. Does the code attempt nested job assignment or skip it, given that message #10 reports the code and its comment disagree?
5. What happens when pg_ctl or a service manager has already placed the postmaster in a job, and which message does the server log?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Andres Freund reviewed and Thomas Munro supports the approach, but Thomas says in message #6 he is not a Windows person. The owner needs to know Windows job and handle semantics.
- **Liability:** The owner signs for Windows process behaviour under service managers, CI runners and nested jobs, and for helper programs that the job now kills. This lasts as long as Windows is supported.
- **Author follow-up:** The author added the requested test in v2 and replied up to message #9. No reply to the review points in message #10 is visible.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is about 200 lines and Windows-only, and the remaining cases (suspended children, loops, spinlock holds) affect a small share of Windows users. Review needs Windows job knowledge, and progress waits on a new version that answers message #10.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Orphan protection silently stops if any other handle to the job exists. Refuted (no failure scenario): The named-job half is a duplicate of security-1. The other half needs a hypothetical later patch: backends already start with handle inheritance, so the v2 test in 013_crash_restart.pl would fail if the job handle were inheritable.
- `maintenance-fit-2` Windows-only mechanism with an unanswered review and no Windows owner. Refuted (wrong facts): Thomas Munro is not the only engaged committer: Andres Freund reviewed in messages #2 and #4, and Thomas plans to build on the patch. The unanswered message #10 is a committability note, which the rubric keeps apart from the verdict.

### Review angles

- prior-attempts: 0 objection(s). No precedent's recorded reason applies here. P2 was returned because deferring exit makes systems fail to shut down, and this proposal does the opposite by terminating children. P6 concerned slow shutdown while the postmaster is alive, which commit 82233ce7e had already solved. Message #8 links an earlier thread that, as message #8 reports it, concluded preemptive termination is the only approach that makes sense. That supports this proposal rather than arguing against it.
- standards-compat: 1 objection(s). Other contract points look fine. There is no new syntax and no new GUC, and nothing changes for dumps or psql. Message #7 settled back-patching, since the author targets a major release.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no on-disk format, WAL record, catalog, wire protocol or extension API, so pg_upgrade and standbys are unaffected. Forced kills after postmaster death match the existing crash-restart model, where WAL replay restores consistency. Not checked: whether archive_command or restore_command programs killed with the job can leave a partial file in the WAL archive.
- security: 1 objection(s). The only security exposure is the job object's name and access rights. No new SQL-level privilege, GUC or data exposure is involved.
- maintenance-fit: 2 objection(s). The core idea fits core Postgres and adds no cost on the hot path. It runs once at postmaster start and only on Windows, and the only committer in the thread supports the approach.
- already-exists: 0 objection(s). Postgres has no existing way to end children that are still suspended, spinning, or holding a spinlock when the postmaster dies. The PostmasterHandle event needs the child to run and check it, so it cannot do the job. A child or external tool cannot do it either without the postmaster creating the job, so this has to be in core. Not checked without the code: how this interacts with any job object that pg_ctl creates in service mode.
- demand: 1 objection(s). Demand is otherwise adequate for a Windows bug fix. Thomas Munro researched the same approach on his own and plans to build on it. The cost falls only on Windows builds, and the patch falls back to current behaviour when job assignment fails.

### Precedents retrieved

- P1: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - used in the review; reason summarised from its thread
- P2: [Do not exit on postmaster death event inside CRIT sections.](https://commitfest.postgresql.org/patch/5964/) (cf, Returned with feedback, 2025-08-28) - used in the review; reason summarised from its thread
- P3: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant; reason summarised from its thread
- P4: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P5: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P6: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - used in the review; reason summarised from its thread
- P7: [Revert "Get rid of WALBufMappingLock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c13070a27b63d9ce4850d88a63bf889a6fde26f0) (revert, Reverted, 2025-08-22) - judged not relevant
- P8: [Revert "Replace PostmasterRandom() with a stronger way of generating randomness."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=faae1c918e8aaae034eaf3ea103fcb6ba9adc5ab) (revert, Reverted, 2016-10-18) - judged not relevant
- P9: [Merging statistics from children instead of re-sampling everything](https://commitfest.postgresql.org/patch/3052/) (cf, Returned with feedback, 2022-04-08) - judged not relevant; reason summarised from its thread
- P10: [Improving the latch handling between logical replication launcher and worker processes.](https://commitfest.postgresql.org/patch/5095/) (cf, Returned with feedback, 2024-10-07) - judged not relevant; reason summarised from its thread
- P11: [base backup client as auxiliary backend process](https://commitfest.postgresql.org/patch/2192/) (cf, Returned with feedback, 2020-02-28) - judged not relevant; reason summarised from its thread
- P12: [Fix Windows network event handling](https://commitfest.postgresql.org/patch/3523/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P13: [Revert commits 290acac92b and 8a7e9e9dad.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=98a9b37ba70f24b28478360d9cf7f190b0f75f8d) (revert, Reverted, 2019-11-30) - judged not relevant
- P14: [Fix cleanup of WSA events in WIN32 wait event code](https://commitfest.postgresql.org/patch/904/) (cf, Rejected, 2016-12-27) - judged not relevant
- P15: [Revert "Close stdin where it's not needed in TestLib.pm procedures"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ca266a069a20c32a8f0a1df982a5a67d9483bcb3) (revert, Reverted, 2019-11-27) - judged not relevant
- P16: [Revert "Ignore object files generated by ecpg test suite on Windows"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2093eb4d4c2b55962df613fdd6f605ce4dc67217) (revert, Reverted, 2015-03-09) - judged not relevant
- P17: [Key management with tests](https://commitfest.postgresql.org/patch/2925/) (cf, Withdrawn, 2021-07-15) - judged not relevant
- P18: [Optional message to signalled process in pg_terminate_backend()/pg_cancel_backend()](https://commitfest.postgresql.org/patch/1174/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P19: [Revert "graceful shutdown" changes for Windows.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=29992a6a509b256efc4ac560a1586b51a64b2637) (revert, Reverted, 2022-03-22) - judged not relevant
- P20: [VACUUM memory management](https://commitfest.postgresql.org/patch/2374/) (cf, Returned with feedback, 2020-07-02) - judged not relevant
- P21: [Improve logrotate test so that it meaningfully exercises syslogger.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d5c22d28f8c161aa729d841aeec936c646044d4) (revert, Partially reverted, 2019-05-19) - judged not relevant
- P22: [Improve handling of OOM errors in libpq making process hangling for COPY and bind (take 2)](https://commitfest.postgresql.org/patch/298/) (cf, Returned with feedback, 2016-02-08) - judged not relevant
- P23: [orphaned function](https://commitfest.postgresql.org/patch/2850/) (cf, Returned with feedback, 2021-04-08) - judged not relevant
- P24: [KMS - Internal key management system](https://commitfest.postgresql.org/patch/2196/) (cf, Returned with feedback, 2020-09-30) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1a objection (demand-1); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 160 s; cost reported by the CLI: $2.18
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 3 of 24 kept
- Tone check: 73 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:45:07.476Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
