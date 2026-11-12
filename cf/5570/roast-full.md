# Design pre-review: Bump soft open file limit (RLIMIT_NOFILE) to hard limit on startup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 5570](https://commitfest.postgresql.org/patch/5570/) (Needs review); authors: Jelte Fennema-Nio (jeltef)
- Thread: [Bump soft open file limit (RLIMIT_NOFILE) to hard limit on startup](https://www.postgresql.org/message-id/flat/CAGECzQQh6VSy3KG4pN1d%3Dh9J%3DD1rStFCMR%2Bt7yh_Kwj-g87aLQ%40mail.gmail.com), started 2025-02-11
- Messages read: 35 of 35
- Patch prose read: v3-0003-Reflect-the-value-of-max_safe_fds-in-max_files_pe.patch, v3-0002-Bump-postmaster-soft-open-file-limit-RLIMIT_NOFIL.patch, v3-0001-Bump-pgbench-soft-open-file-limit-RLIMIT_NOFILE-t.patch, v1-0001-Bump-soft-open-file-limit-RLIMIT_NOFILE-to-hard-l.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No level fails. Message #1 shows demand, and Andres Freund, Tom Lane and the reviewer accept the policy of raising the soft limit to already-open FDs plus max_files_per_process. Two fixable design points survive: standards-compat-2 (children started outside pg_system()/pg_popen() inherit the raised limit, also in the default configuration) and standards-compat-1 (an existing max_files_per_process value now allows more FDs per backend). A committer who knows fd.c and process startup has to judge both.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: The v12 patch prose and documentation hunk were not shown, so the final max_files_per_process wording, the io_uring FD ordering, and the syscall-only child path between fork and exec could not be checked.

## The proposal, restated

**Problem.** The default soft RLIMIT_NOFILE of 1024 is low, and users often fail to raise it because the effective setting depends on how Postgres is started, which cloud users often cannot control. io_uring-based AIO needs one FD per possible child process, and a future threaded server would share one FD limit across all sessions.

**Proposed design.** At startup, set_max_safe_fds() counts the FDs already open and raises the soft limit, up to the hard limit, so that backends can open max_files_per_process files on top of those. Children started by system() and popen() go through new pg_system()/pg_popen() wrappers, which restore the original soft limit after fork and before exec, for programs that use select(2). A second patch copies max_safe_fds back into max_files_per_process.

**User-visible surface.**
- max_files_per_process changes meaning: it now counts files a backend may open beyond the FDs already open, and the docs drop the advice to lower it on "Too many open files".
- The postmaster raises its soft RLIMIT_NOFILE when needed, up to the hard limit.
- Shell commands such as archive_command, restore_command and COPY FROM PROGRAM run with the original soft limit.
- max_files_per_process shows the computed max_safe_fds after startup, possibly as an override source.
- New C functions pg_system() and pg_popen() in fd.c for extensions and core callers that run shell commands.

**What the author claims.**
- Opting in by raising the soft limit at startup is the practice recommended by the systemd author and used by Go and many Debian packages.
- Many user reports show people raising ulimit or lowering max_files_per_process to avoid "Too many open files".
- The change is needed for io_uring AIO and for a future threaded server.
- The default configuration does not use many more FDs, since max_files_per_process stays at 1000.
- Restoring the limit only in the child after fork means the server process never runs with a too-low limit.

**Thread state.** The pgbench part was committed for PG18. Surya Poondla supports the policy and asked to keep the child path between fork and exec at syscall level; Jelte says v12 addresses those three points, and nobody has replied since.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-02-11; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Default Linux installs raise the limit for children outside pg_system/pg_popen

fixable · level 2, rubric item 7 · design, security / privileges, ABI / extension API · `standards-compat-2`

With default settings on Linux, already-open FDs plus 1000 plus reserved FDs exceed 1024, so the postmaster raises the soft limit. A plpython function or an extension then starts a helper through its own fork or subprocess call. That helper inherits the raised limit, and a select(2)-based helper can get FDs at 1024 or above and misbehave.

Tom Lane asked about plperl and plpython in message #4. The material says only that core system() and popen() callers move to pg_system() and pg_popen(). The brief's claim that the default configuration is unaffected covers FD usage, but the soft limit can still be raised in the default case. Extensions that call system() directly keep the old call and get no reset.

Thread: [#4 (Tom Lane, 2025-02-11)](https://www.postgresql.org/message-id/3208430.1739304022%40sss.pgh.pa.us), [#29 ("Jelte Fennema-Nio", 2025-12-07)](https://www.postgresql.org/message-id/DES3UPNP0UXY.35SEHGJ8F8OEQ%40jeltef.nl)

**Question:** Does the default configuration raise the soft limit on Linux, and how do PL language and extension child processes get the original limit back?

Counter-check: held. Message #4 asked about plperl and plpython, and no later message or patch prose answers it. With the new meaning, 1000 files on top of the already-open FDs can exceed 1024, so Tom Lane's relief in message #4 about the default no longer holds.

### 2. Same max_files_per_process value lets backends open more files after upgrade

fixable · level 2, rubric item 12 · new GUC, backpatchability · `standards-compat-1`

A BSD admin once lowered max_files_per_process to stop "Too many open files" and keeps the same postgresql.conf through pg_upgrade. Each backend can now open that many files on top of the inherited and reserved FDs, so the system-wide total grows with max_connections. The admin sees the old errors return and finds that the docs no longer say to lower the setting.

The v3 commit message says max_safe_fds changes from max_files_per_process minus already-open minus NUM_RESERVED_FDS to max_files_per_process itself. That silently changes what an existing configured value means. The documentation hunk drops the advice to lower the setting but adds no release-note or upgrade guidance. Tom Lane in message #7 described Postgres running the kernel out of FDs, which is the case this tuning was meant to prevent.

Thread: [#2 (Tom Lane, 2025-02-11)](https://www.postgresql.org/message-id/3203865.1739301613%40sss.pgh.pa.us), [#7 (Tom Lane, 2025-02-11)](https://www.postgresql.org/message-id/3209989.1739305120%40sss.pgh.pa.us), [#8 (Andres Freund, 2025-02-11)](https://www.postgresql.org/message-id/7u7dbn6s2i6bf3hjzkbqaexj2bpoblqxwbkffbetl4rjv6dcom%40s2uickjc5z53)

**Question:** How will users who tuned max_files_per_process for the old meaning learn the new meaning on upgrade, and what value keeps their old FD budget?

Counter-check: held. The v3 commit message confirms the same value now gives each backend more FDs, and the docs hunk drops the advice to lower it. The increase is only the already-open plus reserved FDs, and release notes are not the patch's job, so this is a docs wording issue.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | no |  |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | New pg_system() and pg_popen() in fd.h, which extensions must call to get the original soft limit in child processes. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but max_files_per_process changes meaning and shows the computed max_safe_fds after startup. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The postmaster changes its own resource limit, and children of PL languages and extensions inherit the raised limit. |
| backpatchability | **yes** | The changed meaning of max_files_per_process makes this a behaviour change for new major versions only. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Core gets its own fork/exec replacement for system() and popen(), and every new caller must use it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. With default settings on Linux, does the postmaster raise the soft limit above 1024, given that 1000 files now come on top of the already-open FDs?
2. How do child processes started by plperl, plpython or an extension calling system() directly get the original soft limit back?
3. What catches a new direct system() or popen() call in core or contrib that bypasses pg_system() and pg_popen()?
4. What value of max_files_per_process keeps the old FD budget for a user who lowered it under the old meaning, and where do the docs say so?
5. How does v12 count io_uring FDs created before set_max_safe_fds(), and what does the patch do on macOS and Windows?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Tom Lane and Andres Freund shaped the design early in the thread, but neither has replied to v12. The owner needs to know fd.c, postmaster startup and child process handling across platforms.
- **Liability:** The owner signs for the fork/exec replacement in pg_system()/pg_popen() on every supported platform, and for any child process that misbehaves under a raised limit, for as long as the code exists.
- **Author follow-up:** Good sign: the author has kept the patch alive through 9 CommitFests, reached v12, and answered the reviewer's three points in November 2026.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Message #1 cites several independent user reports, and io_uring AIO needs the FDs. The patch is small and mostly in fd.c, but the fork/exec path and the GUC meaning change need careful review; no committer has responded to v12 yet.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` Raised limit leaks to any child not started via pg_system()/pg_popen(). Refuted (not specific): Duplicate of standards-compat-2, which covers the same children outside pg_system()/pg_popen() and adds the default-configuration point. The request for a check on direct system() calls can be folded into that one.
- `maintenance-fit-2` Core takes on its own popen() replacement. Refuted (already handled): Message #34 concluded that a small fork/exec is the only workable approach and asked for syscall-only child work plus a comment, and message #35 says v12 does both. The deadlock needs a future threaded server and a future careless change, and ownership belongs under committability.
- `maintenance-fit-3` io_uring descriptor ordering is still open despite being the main motivation. Refuted (no failure scenario): The scenario describes what happens today without the patch, since ring creation before set_max_safe_fds() already runs under the original soft limit. The patch does not make it worse, and the user reports in message #1 justify it without io_uring; the v12 ordering is a fair question to ask.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed row, [P1], reverted a test that set max_files_per_process very low, and its reason does not apply to raising the soft RLIMIT_NOFILE. Message #1 links older list threads [5], [6] and [8], but only as user reports of "Too many open files", not as earlier attempts at this change that were rejected.
- standards-compat: 2 objection(s). Two objections from the compatibility angle. The override of the administrator's soft limit, raised in message #6, was settled in messages #8 and #12, so it is not raised again here.
- replication-ondisk: 0 objection(s). The proposal changes no persistent or cross-node state: no on-disk format, WAL record, catalog, wire protocol or changed struct. restore_command and archive_command run with the original soft limit as before, and pg_system()/pg_popen() are new functions, not changed signatures, so no ABI break follows.
- security: 0 objection(s). The proposal changes no privilege boundary: only the server owner sets max_files_per_process, the raised soft limit is capped by max_files_per_process and the hard limit, and shell commands such as COPY FROM PROGRAM (already gated by pg_execute_server_program) keep the original limit. A session lowering the limit with setrlimit() needs untrusted PL code, which is already superuser-only, and kernel FD exhaustion stays bounded by max_files_per_process as settled in messages #4 and #8.
- maintenance-fit: 3 objection(s). The basic policy of raising the soft limit to already-open descriptors plus max_files_per_process has support from Andres Freund, Tom Lane and the reviewer, so it is not challenged here.
- already-exists: 0 objection(s). No existing feature does this job. Setting LimitNOFILE in a systemd unit or running ulimit -n before pg_ctl does not help users who cannot control how Postgres starts, and cannot cover io_uring FDs that the postmaster creates itself. pg_system() and pg_popen() wrap system() and OpenPipeStream() rather than duplicating them, and P1 concerns a reverted test, not this design.
- demand: 0 objection(s). The material shows real demand. Message #1 cites several independent field reports of "Too many open files" and of raising ulimit. Message #8 says cloud users often cannot change how Postgres starts, and message #10 says io_uring AIO needs one FD per possible child process. Andres Freund, Tom Lane, Tomas Vondra and Surya Poondla accept the policy, and only the mechanism is still debated.

### Precedents retrieved

- P1: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - used in the review
- P2: [Revert "Use "transient" files for blind writes, take 2".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9bacf0e3734f0b5dc821abf7cb312377876e22d3) (revert, Reverted, 2012-10-17) - judged not relevant
- P3: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P4: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P5: [doc: Add some OID/TOAST-related limitations to the limits appendix.](https://commitfest.postgresql.org/patch/4320/) (cf, Withdrawn, 2023-06-13) - judged not relevant; reason summarised from its thread
- P6: [New GUC prepared_statement_limit to limit memory used by prepared statements](https://commitfest.postgresql.org/patch/2241/) (cf, Returned with feedback, 2019-11-28) - judged not relevant; reason summarised from its thread
- P7: [In INSERT/UPDATE, use the table's real tuple descriptor as target.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=20d3fe9009ddbbbb3da3a2da298f922054b43f8c) (revert, Partially reverted, 2020-10-26) - judged not relevant
- P8: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P9: [Revert "Add soft error handling to some expression nodes"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c8ec5e0543b90372c8e6d5cc2cd3d2ff89ca0e82) (revert, Reverted, 2023-10-02) - judged not relevant
- P10: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - judged not relevant; reason summarised from its thread
- P11: [Rearrange execution of PARAM_EXTERN Params for plpgsql's benefit.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6719b238e8f0ea83c39d2b1728c7670cdaa34e06) (revert, Partially reverted, 2017-12-21) - judged not relevant
- P12: [add max_worker_processes_per_user setting](https://commitfest.postgresql.org/patch/1253/) (cf, Returned with feedback, 2017-11-30) - judged not relevant; reason summarised from its thread
- P13: [Adding linux cachestat in file Readv and Prefetch](https://commitfest.postgresql.org/patch/4779/) (cf, Rejected, 2024-03-14) - judged not relevant; reason summarised from its thread
- P14: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P15: [Revert "Do execGrouping.c via expression eval machinery."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=2a41507dab0f293ff241fe8ae326065998668af8) (revert, Reverted, 2018-02-16) - judged not relevant
- P16: [rand48 replacement](https://commitfest.postgresql.org/patch/3132/) (cf, Withdrawn, 2021-05-24) - judged not relevant
- P17: [Move bki file pre-processing from initdb to bootstrap](https://commitfest.postgresql.org/patch/4543/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [rich constructor for plpy.Error class, possible raise structured exception](https://commitfest.postgresql.org/patch/387/) (cf, Returned with feedback, 2016-02-03) - judged not relevant
- P19: [Revert recent changes related to handling of 2PC files at recovery](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=a6c70f68cdeb3bed591a2dd7d421bf3536ba5216) (revert, Reverted, 2025-01-17) - judged not relevant
- P20: [Avoid possibly slow processing during postmaster abnormal or immediate shutdown](https://commitfest.postgresql.org/patch/816/) (cf, Rejected, 2016-11-18) - judged not relevant
- P21: [Startup Centralization](https://commitfest.postgresql.org/patch/2259/) (cf, Returned with feedback, 2019-12-01) - judged not relevant
- P22: [Revert "Hack pg_ctl to report postmaster's exit status."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f38291e927fa8c04eb772e6a17a3dd44da2b69e8) (revert, Reverted, 2019-10-13) - judged not relevant
- P23: [Replace "implementor" with "implementer"](https://commitfest.postgresql.org/patch/3957/) (cf, Withdrawn, 2022-10-23) - judged not relevant
- P24: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 15; wall time: 124 s; cost reported by the CLI: $3.00
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 4 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 73 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:33:14.503Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
