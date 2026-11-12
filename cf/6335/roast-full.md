# Design pre-review: File locks for data directory lockfile in the context of Linux namespaces

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6335](https://commitfest.postgresql.org/patch/6335/) (Needs review); authors: Dmitry Dolgov (erthalion)
- Thread: [File locks for data directory lockfile in the context of Linux namespaces](https://www.postgresql.org/message-id/flat/z2u3jtjzhqofqhrjvfgkwl4cczhpqibz3t47gpx7fbe7ceaihk%40i5mveu2wfpxn), started 2025-12-19
- Messages read: 10 of 10
- Patch prose read: v5-0001-Use-open-file-description-locks-for-lockfiles.patch, v1-0001-Use-open-file-description-locks-for-data-director.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is stated, reproduced in message #3, and no surviving objection disputes the goal. Two fixable points remain. replication-ondisk-2 asks why a new startup FATAL should go into minor releases. standards-compat-1 shows that a child of archive_command or restore_command can keep the inherited lock and block a restart. Both need a committer who knows the startup interlock to decide.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 10 messages and no code were read. How v5 handles the lock fd in child processes, the stale-lockfile path and platforms without F_OFD_SETLK could not be checked.

## The proposal, restated

**Problem.** Two postmasters in different Linux PID namespaces can start on the same data directory, because each sees its own PID in postmaster.pid and treats the file as stale. The author reports this happens with Kubernetes ReadWriteOnce volumes after a forced pod termination. It affects container users who share a data volume across pods.

**Proposed design.** At startup Postgres takes an exclusive fcntl F_OFD_SETLK lock on each lockfile, on top of the existing PID check. If the lock is held, startup fails. If locking fails for another reason, Postgres logs a warning and continues as before. The lock fd is kept open without FD_CLOEXEC, so child processes inherit it.

**User-visible surface.**
- New FATAL at startup: cannot lock the lock file "postmaster.pid", with HINT "Another server is starting."
- Unix socket lockfiles (.s.PGSQL.nnnn.lock) are also OFD-locked, so a second postmaster on the same socket now fails to start.
- New WARNING when the lock call fails with an unexpected error.
- New configure and meson check for F_OFD_SETLK.

**What the author claims.**
- F_OFD_SETLK is now part of POSIX.1 2024, which removes the past portability objection.
- The change fixes the namespace problem everywhere except NFSv3, where OFD locks become process-associated advisory locks.
- On NFSv4 the author found no sign that locks are converted to advisory locks.
- The scenario happens in the field with Kubernetes, mostly through misconfiguration.
- Leaving out FD_CLOEXEC is intended, so the lock lasts as long as any process that may touch the data directory.

**Thread state.** Ilmar Yunusov confirmed the namespace fix on Linux for v2 and v3, and Zsolt Parragi raised error-handling points on v4. v5 is posted with no reply yet, and the leak on the stale path and the elog versus ereport choice are still disputed.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-12-19; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Stable target changes startup interlock behaviour in a minor release

fixable · level 1b, rubric item 6 · backpatchability · `replication-ondisk-2`

A site applies a minor update and restarts during a hung restore_command on a standby. The restart worked on the previous minor release and now fails at the new FATAL. The same can happen where a container wrapper leaks the inherited fd. The site learns of the change only from a failed restart.

The CommitFest entry targets stable, but the thread has no discussion of back-patching. The patch adds a configure and meson check, a new FATAL path for postmaster.pid and the socket lockfiles, and new fd lifetime rules. A committer would likely want that in master first, with a release-notes entry, before releasing it to existing clusters.

Thread: [#1 (Dmitry Dolgov, 2025-12-19)](https://www.postgresql.org/message-id/z2u3jtjzhqofqhrjvfgkwl4cczhpqibz3t47gpx7fbe7ceaihk%40i5mveu2wfpxn), [#9 (Dmitry Dolgov, 2026-07-10)](https://www.postgresql.org/message-id/uljhsglx5lo4ox3xxkjoov4ye7l3s6xh66gw4gfysgxpla7vgk%40fl6arpructd2)

**Question:** Why should this ship in back branches rather than master only, given that it adds new startup failures for existing clusters?

Counter-check: held. The CommitFest entry says target "stable", and no message discusses back-patching. A restart during a hung restore_command works on current minor releases and would fail with the new FATAL, so the request for master only stands.

### 2. Inherited lock fds can block restart while unrelated child processes live

fixable · level 2, rubric item 14 · design, backpatchability, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An archive_command, restore_command or COPY ... TO PROGRAM starts a long rsync or a background daemon, then the postmaster crashes or is killed. The child still holds the inherited fd and so the OFD lock, and restart fails with FATAL "cannot lock the lock file" and HINT "Another server is starting." while no server runs.

Message #8 asked for FD_CLOEXEC, and message #9 kept the fd inheritable because "the lock file is created by the bootstrap process and the postmaster". That answer covers Postgres processes but not shell children, which do not attach to shared memory and so do not block restart today. The proposal adds a new way for startup to fail, with a HINT that points the DBA at the wrong cause.

Thread: [#8 (Zsolt Parragi, 2026-07-06)](https://www.postgresql.org/message-id/CAN4CZFMepY-70PqQjefnFFR1VFjvos5EnkpU75ho9bvQj%3DKXoQ%40mail.gmail.com), [#9 (Dmitry Dolgov, 2026-07-10)](https://www.postgresql.org/message-id/uljhsglx5lo4ox3xxkjoov4ye7l3s6xh66gw4gfysgxpla7vgk%40fl6arpructd2)

**Question:** Which processes should hold the lock after exec, and what does a DBA see and do when an orphaned archive_command or TO PROGRAM child keeps it?

Counter-check: held. Children started through system() inherit an fd without FD_CLOEXEC, and they detach from shared memory at exec, so they do not block a restart today. Message #9 only discusses the bootstrap process and the postmaster, and the HINT would name the wrong cause.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | Children of archive_command and restore_command inherit the lock fd and can block a restart of a primary or standby. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CommitFest entry targets stable, and the patch adds a new startup FATAL and new configure and meson checks. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second interlock in miscinit.c behaves differently by platform and filesystem: Linux local filesystems, NFSv3, NFSv4, systems without F_OFD_SETLK. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which processes should hold the lock after exec, and what does a DBA do when an orphaned archive_command, restore_command or COPY ... TO PROGRAM child keeps it?
2. Is the patch meant for back branches, and if so, why should a new startup FATAL ship in minor releases and not only in the next major version?
3. What does the patch do on platforms without F_OFD_SETLK, such as Windows or older BSD and macOS releases?
4. Has anyone tested the lock on NFSv3 and NFSv4, and does it ever report a false conflict there?
5. Does v5 fix the Windows build failure from the LockFile typedef name clash?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the postmaster startup interlock in miscinit.c and file locking behaviour across platforms and network filesystems.
- **Liability:** The owner signs for the startup interlock on every platform and filesystem, including false lock conflicts that stop a server from starting, with no end date.
- **Author follow-up:** The author posted five versions over about eleven months and answered both reviewers, which suggests he stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is about 13 KiB in one backend file plus build checks, and it prevents two postmasters on one data directory in container setups. It has been in five CommitFests with two reviewers and no committer in the visible thread.

## Appendix

### Refuted in the counter-check

- `standards-compat-2` Stable target adds a new startup failure mode in minor releases. Refuted (not specific): Duplicate of replication-ondisk-2. Its shared socket directory scenario is also weak, because two postmasters on the same port and socket directory already conflict outside the namespace case.
- `replication-ondisk-1` Inherited lock fd lets surviving children block postmaster restart. Refuted (not specific): Duplicate of standards-compat-1, which states the same inherited fd scenario more briefly and adds the misleading HINT.
- `maintenance-fit-1` Inherited lock fd lets non-server children block a restart. Refuted (not specific): Duplicate of standards-compat-1. Its statement that these children never touch the data directory is also inexact, since archive_command reads WAL files from it.
- `maintenance-fit-2` Stable target adds a new startup failure in minor releases. Refuted (not specific): Duplicate of replication-ondisk-2. Its example of a second postmaster sharing a socket directory that "worked before" describes only the broken namespace case the patch is meant to stop.
- `maintenance-fit-3` Second interlock with per-platform behaviour and no committer engaged. Refuted (no failure scenario): The scenario is a possible future bug report on an unnamed platform, and where the lock is unavailable the design falls back to today's behaviour. Committer silence and missing tests belong to committability and level 4, and v5 followed the open points in message #9.

### Review angles

- prior-attempts: 0 objection(s). No precedent found in the list, and message #1 and the commit message name only portability and NFS as past reasons, which the best-effort design on top of the PID check addresses. The thread gives no links to those earlier discussions, so other past reasons cannot be checked from the material.
- standards-compat: 2 objection(s). No SQL, GUC, pg_dump or psql surface is involved. The objections cover lock lifetime and the stable target only.
- replication-ondisk: 2 objection(s). The patch changes no WAL, catalog, on-disk relation format or wire protocol. Its persistent state is the lockfiles and the lifetime of the inherited fds.
- security: 0 objection(s). The proposal adds no new privilege or exposure. The lockfiles are created mode 0600 by the server OS user, so no other local user can open them and hold the OFD lock. Children that inherit the lock fd, such as archive_command or COPY PROGRAM, already run as the server OS user and could open postmaster.pid anyway, so they gain no access. A long-lived inherited fd blocking restart is an availability and design question for another angle. If locking fails, the server behaves as it does today.
- maintenance-fit: 3 objection(s). Three objections from the maintenance angle. None is fatal. The basic idea, a best-effort OFD lock on top of the PID check, is small and is limited to startup.
- already-exists: 0 objection(s). No existing Postgres feature closes this gap. The postmaster.pid PID check and the SysV shared memory check in the lockfile are both scoped to a namespace, so a postmaster in a separate PID and IPC namespace passes both, as message #3 shows. An entrypoint wrapper such as flock(1) could take the same lock outside core, but only users who know about the risk would add it. The users in message #1 hit the problem through misconfiguration, so they would not add it. That makes the wrapper a partial idea rather than an equal alternative. A Kubernetes access mode is not raised here, because I cannot confirm that it covers the forced-termination case.
- demand: 0 objection(s). No demand objection survives. The failure is two postmasters writing one data directory, which corrupts data. Message #1 reports it in the field with Kubernetes ReadWriteOnce volumes, and the reviewer reproduced it in message #3. The fix adds no GUC or syntax, and users without the problem see no change unless another server holds the lock. The thread gives no count of field reports, but for a cheap interlock that prevents corruption that would not justify a 'not now'.

### Precedents retrieved

- P1: [Revert "pg_resetwal: Improve error with wrong/missing data directory"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5daa50f962bd06e49af690efc2a8a27fdd37fb7d) (revert, Reverted, 2023-09-29) - judged not relevant
- P2: [pg_basebackup to adjust existing data directory permissions](https://commitfest.postgresql.org/patch/2018/) (cf, Returned with feedback, 2019-11-27) - judged not relevant; reason summarised from its thread
- P3: [Revert exporting of internal GUC variable "data_directory".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=da33a3894e0fc440ac53cdc0f2e360e703b13b8c) (revert, Reverted, 2015-05-29) - judged not relevant
- P4: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant; reason summarised from its thread
- P5: [psql BACKEND_PID variable](https://commitfest.postgresql.org/patch/4165/) (cf, Rejected, 2023-03-18) - judged not relevant; reason summarised from its thread
- P6: [Remove pgpid_t type, use pid_t instead](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=5035c93c8a5ac6804da79c67403460348b381924) (revert, Partially reverted, 2022-10-22) - judged not relevant
- P7: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1133/) (cf, Returned with feedback, 2018-03-30) - judged not relevant; reason summarised from its thread
- P8: [Moving relation extension locks out of heavyweight lock manager](https://commitfest.postgresql.org/patch/1666/) (cf, Returned with feedback, 2018-12-01) - judged not relevant; reason summarised from its thread
- P9: [Revert "Display the time when the process started waiting for the lock, in pg_locks."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=890d2182a2c425aaa80f9bf9f7116d31e0c6538e) (revert, Reverted, 2021-02-09) - judged not relevant
- P10: [Patch for migration of the pg_commit_ts directory](https://commitfest.postgresql.org/patch/5710/) (cf, Withdrawn, 2025-10-15) - judged not relevant; reason summarised from its thread
- P11: [Speed up LockReleaseAll](https://commitfest.postgresql.org/patch/3220/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P12: [Lockable views](https://commitfest.postgresql.org/patch/1334/) (cf, Returned with feedback, 2017-11-29) - judged not relevant; reason summarised from its thread
- P13: [pg_receivewal fail to streams when the partial file to write is not fully initialized present in the wal receiver directory](https://commitfest.postgresql.org/patch/3503/) (cf, Returned with feedback, 2024-01-22) - judged not relevant; reason summarised from its thread
- P14: [Find additional connection service files in pg_service.conf.d directory](https://commitfest.postgresql.org/patch/1480/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P15: [LOCK TABLE .. DEFERRABLE](https://commitfest.postgresql.org/patch/602/) (cf, Returned with feedback, 2016-12-02) - judged not relevant
- P16: [notice lock waits](https://commitfest.postgresql.org/patch/702/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P17: [Unified file access using virtual file descriptors](https://commitfest.postgresql.org/patch/4420/) (cf, Returned with feedback, 2024-02-01) - judged not relevant
- P18: [Reuse Workers and Replication Slots during Logical Replication](https://commitfest.postgresql.org/patch/3784/) (cf, Returned with feedback, 2024-01-10) - judged not relevant
- P19: [Revert 9.5 pgindent changes to atomics directory files](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=befa3e648ce018d84cd2a0df701927c56fe3da4e) (revert, Reverted, 2015-05-25) - judged not relevant
- P20: [Contains and is contained by operators of inet datatypes](https://commitfest.postgresql.org/patch/805/) (cf, Rejected, 2016-11-30) - judged not relevant
- P21: [some namespace.c refactoring](https://commitfest.postgresql.org/patch/4212/) (cf, Returned with feedback, 2024-01-26) - judged not relevant
- P22: [Add OID descriptions to dumped parse/query/plan trees](https://commitfest.postgresql.org/patch/5987/) (cf, Withdrawn, 2025-08-25) - judged not relevant
- P23: [Revert test added by commit d207038053.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=ed7bb5c311ce059294807cf4abfa9406d95feec0) (revert, Reverted, 2020-01-14) - judged not relevant
- P24: [Doc: Improve hostssl related descriptions and option presentation](https://commitfest.postgresql.org/patch/5088/) (cf, Withdrawn, 2025-02-27) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (replication-ondisk-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 16; wall time: 143 s; cost reported by the CLI: $2.02
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 82 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:49:08.892Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
