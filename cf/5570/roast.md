# Design pre-review: Bump soft open file limit (RLIMIT_NOFILE) to hard limit on startup

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 5570](https://commitfest.postgresql.org/patch/5570/); [thread](https://www.postgresql.org/message-id/flat/CAGECzQQh6VSy3KG4pN1d%3Dh9J%3DD1rStFCMR%2Bt7yh_Kwj-g87aLQ%40mail.gmail.com), 35 of 35 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No level fails. Message #1 shows demand, and Andres Freund, Tom Lane and the reviewer accept the policy of raising the soft limit to already-open FDs plus max_files_per_process. Two fixable design points survive: standards-compat-2 (children started outside pg_system()/pg_popen() inherit the raised limit, also in the default configuration) and standards-compat-1 (an existing max_files_per_process value now allows more FDs per backend). A committer who knows fd.c and process startup has to judge both.

Not checked: The v12 patch prose and documentation hunk were not shown, so the final max_files_per_process wording, the io_uring FD ordering, and the syscall-only child path between fork and exec could not be checked.

## Prior attempts

None found before 2025-02-11.

## Objections a reviewer is likely to raise

### 1. Default Linux installs raise the limit for children outside pg_system/pg_popen

With default settings on Linux, already-open FDs plus 1000 plus reserved FDs exceed 1024, so the postmaster raises the soft limit. A plpython function or an extension then starts a helper through its own fork or subprocess call. That helper inherits the raised limit, and a select(2)-based helper can get FDs at 1024 or above and misbehave.

Tom Lane asked about plperl and plpython in message #4. The material says only that core system() and popen() callers move to pg_system() and pg_popen(). The brief's claim that the default configuration is unaffected covers FD usage, but the soft limit can still be raised in the default case. Extensions that call system() directly keep the old call and get no reset.

**Question:** Does the default configuration raise the soft limit on Linux, and how do PL language and extension child processes get the original limit back?

<sub>fixable, level 2, design; messages [#4](https://www.postgresql.org/message-id/3208430.1739304022%40sss.pgh.pa.us), [#29](https://www.postgresql.org/message-id/DES3UPNP0UXY.35SEHGJ8F8OEQ%40jeltef.nl); `standards-compat-2`</sub>

### 2. Same max_files_per_process value lets backends open more files after upgrade

A BSD admin once lowered max_files_per_process to stop "Too many open files" and keeps the same postgresql.conf through pg_upgrade. Each backend can now open that many files on top of the inherited and reserved FDs, so the system-wide total grows with max_connections. The admin sees the old errors return and finds that the docs no longer say to lower the setting.

The v3 commit message says max_safe_fds changes from max_files_per_process minus already-open minus NUM_RESERVED_FDS to max_files_per_process itself. That silently changes what an existing configured value means. The documentation hunk drops the advice to lower the setting but adds no release-note or upgrade guidance. Tom Lane in message #7 described Postgres running the kernel out of FDs, which is the case this tuning was meant to prevent.

**Question:** How will users who tuned max_files_per_process for the old meaning learn the new meaning on upgrade, and what value keeps their old FD budget?

<sub>fixable, level 2, new GUC; messages [#2](https://www.postgresql.org/message-id/3203865.1739301613%40sss.pgh.pa.us), [#7](https://www.postgresql.org/message-id/3209989.1739305120%40sss.pgh.pa.us), [#8](https://www.postgresql.org/message-id/7u7dbn6s2i6bf3hjzkbqaexj2bpoblqxwbkffbetl4rjv6dcom%40s2uickjc5z53); `standards-compat-1`</sub>

## Questions for the author

1. With default settings on Linux, does the postmaster raise the soft limit above 1024, given that 1000 files now come on top of the already-open FDs?
2. How do child processes started by plperl, plpython or an extension calling system() directly get the original soft limit back?
3. What catches a new direct system() or popen() call in core or contrib that bypasses pg_system() and pg_popen()?
4. What value of max_files_per_process keeps the old FD budget for a user who lowered it under the old meaning, and where do the docs say so?
5. How does v12 count io_uring FDs created before set_max_safe_fds(), and what does the patch do on macOS and Windows?

## Touch points

- **ABI / extension API:** New pg_system() and pg_popen() in fd.h, which extensions must call to get the original soft limit in child processes.
- **new GUC:** No new GUC, but max_files_per_process changes meaning and shows the computed max_safe_fds after startup.
- **security / privileges:** The postmaster changes its own resource limit, and children of PL languages and extensions inherit the raised limit.
- **backpatchability:** The changed meaning of max_files_per_process makes this a behaviour change for new major versions only.
- **maintenance burden ("who maintains this in 10 years"):** Core gets its own fork/exec replacement for system() and popen(), and every new caller must use it.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane and Andres Freund shaped the design early in the thread, but neither has replied to v12. The owner needs to know fd.c, postmaster startup and child process handling across platforms.
- Priority: proceed; audience a sizeable minority; review cost medium. Message #1 cites several independent user reports, and io_uring AIO needs the FDs. The patch is small and mostly in fd.c, but the fork/exec path and the GUC meaning change need careful review; no committer has responded to v12 yet.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
