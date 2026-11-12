# Design pre-review: File locks for data directory lockfile in the context of Linux namespaces

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6335](https://commitfest.postgresql.org/patch/6335/); [thread](https://www.postgresql.org/message-id/flat/z2u3jtjzhqofqhrjvfgkwl4cczhpqibz3t47gpx7fbe7ceaihk%40i5mveu2wfpxn), 10 of 10 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). The problem is stated, reproduced in message #3, and no surviving objection disputes the goal. Two fixable points remain. replication-ondisk-2 asks why a new startup FATAL should go into minor releases. standards-compat-1 shows that a child of archive_command or restore_command can keep the inherited lock and block a restart. Both need a committer who knows the startup interlock to decide.

Not checked: Only 3 of 10 messages and no code were read. How v5 handles the lock fd in child processes, the stale-lockfile path and platforms without F_OFD_SETLK could not be checked.

## Prior attempts

None found before 2025-12-19.

## Objections a reviewer is likely to raise

### 1. Stable target changes startup interlock behaviour in a minor release

A site applies a minor update and restarts during a hung restore_command on a standby. The restart worked on the previous minor release and now fails at the new FATAL. The same can happen where a container wrapper leaks the inherited fd. The site learns of the change only from a failed restart.

The CommitFest entry targets stable, but the thread has no discussion of back-patching. The patch adds a configure and meson check, a new FATAL path for postmaster.pid and the socket lockfiles, and new fd lifetime rules. A committer would likely want that in master first, with a release-notes entry, before releasing it to existing clusters.

**Question:** Why should this ship in back branches rather than master only, given that it adds new startup failures for existing clusters?

<sub>fixable, level 1b, backpatchability; messages [#1](https://www.postgresql.org/message-id/z2u3jtjzhqofqhrjvfgkwl4cczhpqibz3t47gpx7fbe7ceaihk%40i5mveu2wfpxn), [#9](https://www.postgresql.org/message-id/uljhsglx5lo4ox3xxkjoov4ye7l3s6xh66gw4gfysgxpla7vgk%40fl6arpructd2); `replication-ondisk-2`</sub>

### 2. Inherited lock fds can block restart while unrelated child processes live

An archive_command, restore_command or COPY ... TO PROGRAM starts a long rsync or a background daemon, then the postmaster crashes or is killed. The child still holds the inherited fd and so the OFD lock, and restart fails with FATAL "cannot lock the lock file" and HINT "Another server is starting." while no server runs.

Message #8 asked for FD_CLOEXEC, and message #9 kept the fd inheritable because "the lock file is created by the bootstrap process and the postmaster". That answer covers Postgres processes but not shell children, which do not attach to shared memory and so do not block restart today. The proposal adds a new way for startup to fail, with a HINT that points the DBA at the wrong cause.

**Question:** Which processes should hold the lock after exec, and what does a DBA see and do when an orphaned archive_command or TO PROGRAM child keeps it?

<sub>fixable, level 2, design; messages [#8](https://www.postgresql.org/message-id/CAN4CZFMepY-70PqQjefnFFR1VFjvos5EnkpU75ho9bvQj%3DKXoQ%40mail.gmail.com), [#9](https://www.postgresql.org/message-id/uljhsglx5lo4ox3xxkjoov4ye7l3s6xh66gw4gfysgxpla7vgk%40fl6arpructd2); `standards-compat-1`</sub>

## Questions for the author

1. Which processes should hold the lock after exec, and what does a DBA do when an orphaned archive_command, restore_command or COPY ... TO PROGRAM child keeps it?
2. Is the patch meant for back branches, and if so, why should a new startup FATAL ship in minor releases and not only in the next major version?
3. What does the patch do on platforms without F_OFD_SETLK, such as Windows or older BSD and macOS releases?
4. Has anyone tested the lock on NFSv3 and NFSv4, and does it ever report a false conflict there?
5. Does v5 fix the Windows build failure from the LockFile typedef name clash?

## Touch points

- **WAL / replication / standby:** Children of archive_command and restore_command inherit the lock fd and can block a restart of a primary or standby.
- **backpatchability:** The CommitFest entry targets stable, and the patch adds a new startup FATAL and new configure and meson checks.
- **maintenance burden ("who maintains this in 10 years"):** A second interlock in miscinit.c behaves differently by platform and filesystem: Linux local filesystems, NFSv3, NFSv4, systems without F_OFD_SETLK.

## Who would commit it, and when

- Owner: silent. It needs a committer who knows the postmaster startup interlock in miscinit.c and file locking behaviour across platforms and network filesystems.
- Priority: needs champion; audience a sizeable minority; review cost low. The patch is about 13 KiB in one backend file plus build checks, and it prevents two postmasters on one data directory in container setups. It has been in five CommitFests with two reviewers and no committer in the visible thread.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 5 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
