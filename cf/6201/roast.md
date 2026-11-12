# Design pre-review: Fix orphaned backend processes on Windows using Job Objects

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6201](https://commitfest.postgresql.org/patch/6201/); [thread](https://www.postgresql.org/message-id/flat/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is against the idea, and two committers in the thread support terminating children from the kernel. Three fixable points remain. The commit message and docs give a cause that message #7 withdrew (demand-1). Job membership also covers helper programs that are not backends (standards-compat-1). The job's name and access rights are not stated (security-1). Confidence is medium because security-1 rests on an inference from message #10.

Not checked: Without the code, the job's name, security descriptor, handle inheritance and breakaway flags could not be checked. The interaction with a job that pg_ctl creates in service mode is also unchecked.

## Prior attempts

- [Use SIGIO to detect postmaster death](https://www.postgresql.org/message-id/flat/CA+hUKGK1607VmtrDUHQXrsooU=ap4g4R2yaoByWOOA3m8xevUQ@mail.gmail.com) (similar idea; Withdrawn, 2021-07-13). "On closer inspection, only the last backend to start up and do F_SETOWN on the pipe receives the signal." ([message](https://www.postgresql.org/message-id/CA%2BhUKGLua5Pnx%2Brjjxg%3D3OPO4oOjtUPjeUYeZXh7TkH3gA7-Uw%40mail.gmail.com)) Different now: P1 signalled backends on Unix through SIGIO on a shared pipe, and only one backend got the signal. This proposal targets Windows and has the kernel terminate every job member, so that failure does not arise.

## Objections a reviewer is likely to raise

### 1. Stated problem and the reproduction in the commit message do not match

A committer reads the commit message and docs, which say the taskkill /F case orphans backends because existing mechanisms need postmaster code to run at exit. Thomas Munro says PostmasterHandle is signalled by the kernel, and the author agreed. The commit would then record a wrong cause, and later readers cannot tell which cases the patch fixes and which the handle-inheritance patches fix.

Message #7 says the wording of the main reproduction in message #1 was imprecise. The likely real cause is inherited handles, which other CF entries address. The cases left to this patch are children still suspended, infinite loops and spinlock holds (messages #7 and #9). Thomas notes in message #8 that he recalls no recent "manual orphan cleanup needed" reports, so the field-visible problem may be smaller than the commit message says. He also agrees that preemptive termination is the right design.

**Question:** Which failure cases remain once the handle-inheritance patches land, and will the commit message and docs be rewritten to name only those cases?

<sub>fixable, level 1a, problem statement; messages [#1](https://www.postgresql.org/message-id/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), [#6](https://www.postgresql.org/message-id/CA%2BhUKGLVBGE2KkzLaDkKX9t7%3Dt2BvjtOLXef5NnMv4cAZyoz7w%40mail.gmail.com), [#7](https://www.postgresql.org/message-id/f60065bd-bed3-4e78-beb6-7e81d5e3875d%40gmail.com), [#8](https://www.postgresql.org/message-id/CA%2BhUKGJo6hu6GToiXarBRF%2BAqhFPnzMTW2Nksm0x-%2B9m2%3DdskQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/5e80ef94-fa47-4110-9e3a-7007a4ecef14%40gmail.com); `demand-1`</sub>

### 2. Job membership kills detached helper programs, even at clean shutdown

A Windows user's archive_command or COPY PROGRAM starts a detached helper, for example with `start /b`, that should finish an upload after the command returns. After upgrade the helper is still a job member. When the postmaster exits, even cleanly, the kernel kills it mid-upload, and the user finds out from missing or partial files.

The postmaster is assigned to the job, so every descendant inherits membership, not only backends, and KILL_ON_JOB_CLOSE fires on any postmaster exit. Message #1 says clean shutdown leaves "nothing left to kill", but that holds only for processes the postmaster waits for. Today such grandchildren survive the postmaster on Windows, so this changes behaviour for existing scripts and extensions without any request from the user. Neither the documentation hunk nor the thread covers archive_command, restore_command, COPY PROGRAM or extension-spawned processes, and the brief lists this as unclear.

**Question:** Which processes outside the postmaster's own children stay in the job, and will the job allow breakaway or will the documentation state that they are killed on any postmaster exit?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/880214db-ab8c-4b9e-852c-b0f6d90d3f3d%40gmail.com), [#7](https://www.postgresql.org/message-id/f60065bd-bed3-4e78-beb6-7e81d5e3875d%40gmail.com); `standards-compat-1`</sub>

### 3. Named job object can be pre-created or opened by another local process

A local account that is not the service account creates a job object with the expected name before the postmaster starts. CreateJobObject then returns a handle to that existing object, and the other account keeps its own handle. It can call TerminateJobObject to kill every backend, or hold the job open so orphans survive. The DBA sees only unexplained crashes or orphaned backends.

Message #10 adds a separate patch to use an unnamed job, which suggests v2 names its job object. Message #10 also says that naming the job, or leaving it unnamed, does not set the access policy. The patch prose does not give the name, the namespace, the security descriptor, or whether ERROR_ALREADY_EXISTS counts as a failure. A Windows named kernel object with a predictable name and default rights is a known way to take over another process's object, and the author has not answered message #10.

**Question:** Does the postmaster create an unnamed job with a restrictive security descriptor and a non-inheritable handle, and does it refuse a job object that already exists?

<sub>fixable, level 2, security / privileges; messages [#10](https://www.postgresql.org/message-id/CAAH-eSVh63%2B_OwacE8dkhc%2BVZkAwLcScumA-ChrReeodFmDuFw%40mail.gmail.com); `security-1`</sub>

## Questions for the author

1. Which failure cases remain for this patch once the handle-inheritance patches land, and will the commit message and docs name only those cases?
2. Are helper programs started from archive_command, restore_command, COPY PROGRAM or an extension killed when the postmaster exits cleanly, or does the job allow breakaway?
3. Is the job object named, what security descriptor does it get, and does the postmaster refuse a job object that already exists?
4. Does the code attempt nested job assignment or skip it, given that message #10 reports the code and its comment disagree?
5. What happens when pg_ctl or a service manager has already placed the postmaster in a job, and which message does the server log?

## Touch points

- **security / privileges:** The postmaster creates a kernel job object whose name, access rights and handle inheritance the patch prose does not state.
- **backpatchability:** The entry is tagged Bugfix but changes process behaviour on Windows; the author targets a major release only.
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a Windows-only file, pg_job_object.c, that depends on job semantics under service managers and nested jobs.

## Who would commit it, and when

- Owner: qualified committer engaged. Andres Freund reviewed and Thomas Munro supports the approach, but Thomas says in message #6 he is not a Windows person. The owner needs to know Windows job and handle semantics.
- Priority: proceed; audience niche; review cost medium. The patch is about 200 lines and Windows-only, and the remaining cases (suspended children, loops, spinlock holds) affect a small share of Windows users. Review needs Windows job knowledge, and progress waits on a new version that answers message #10.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
