# Design pre-review: Fix pg_restore gzip stream validation (bug #19754)

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7456](https://commitfest.postgresql.org/patch/7456/); [thread](https://www.postgresql.org/message-id/flat/C482E34B-347F-4AD2-ACE1-96142F6088B2%40gmail.com), 5 of 5 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). The bug is real and the reporter confirmed the fix on two platforms with three zlib versions, so nothing fails at level 1. Two fixable design points survive, and both concern the back-patch that message #3 asks for. maintenance-fit-1: v2-0002 bundles the pg_restore fix with a refactor of astreamer_gzip, which other tools use. standards-compat-1: zero-byte files and trailing bytes become errors. A committer who knows the frontend compression code has to decide the back-branch shape.

Not checked: Without the code, the review could not check whether the refactored astreamer_gzip behaves the same for the other tools, or whether memory use stays bounded on concatenated or malformed gzip members.

## Prior attempts

- [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://www.postgresql.org/message-id/flat/20231231.200741.1078989336605759878.horikyota.ntt@gmail.com) (related; Reverted, 2024-01-23). "Since it was non-essential refactoring, let's revert it to make v16 work the same as earlier releases." Different now: P18 was a pure refactor of libpqwalreceiver. This proposal is a bug fix that bundles a refactor of shared frontend code used by other tools.

## Objections a reviewer is likely to raise

### 1. Back-patched bugfix also refactors astreamer_gzip, used by other tools

A packager ships a 16-18 minor release with this patch to fix pg_restore. The astreamer_gzip rework changes how pg_basebackup, pg_verifybackup or pg_waldump handle gzip input. A user finds this only when a backup check or WAL read starts to fail after a minor update.

Message #3 asks for a back-patch to 14, and v2-0002 moves the inflate logic out of astreamer_gzip into a new src/fe_utils/gzip.c. That moves code used by tools that have no reported bug. [P18] reverted a non-essential refactor from a released branch for this reason. 14 and 15 lack compress_gzip.c (message #3), so master's patch shape does not carry over, and the project would keep two or three versions of the fix.

Precedent: [Revert "libpqwalreceiver: Convert to libpq-be-fe-helpers.h"](https://www.postgresql.org/message-id/flat/20231231.200741.1078989336605759878.horikyota.ntt@gmail.com) (Reverted, 2024-01-23)

**Question:** Can the series be split into a minimal back-patchable pg_restore fix and a separate master-only astreamer_gzip refactor, and what does the fix look like for 14 and 15?

<sub>fixable, level 2, backpatchability; messages [#1](https://www.postgresql.org/message-id/C482E34B-347F-4AD2-ACE1-96142F6088B2%40gmail.com), [#3](https://www.postgresql.org/message-id/001b01dd5d3f%24c6a15e90%2453e41bb0%24%40gmx.net); `maintenance-fit-1`</sub>

### 2. Stricter gzip parsing rejects files that pg_restore accepts today

A user keeps a directory-format dump that contains a zero-byte data file, or whose gzip files carry trailing padding from a copy or archive tool. After a minor upgrade that includes this fix, pg_restore stops with an error. The user finds out during a restore, possibly one run to recover lost data.

Message #3 confirms that zero-byte files and trailing bytes after the last member are now errors. The only argument offered is that current pg_dump always writes at least an empty member. That covers newly written dumps, but not dumps from older pg_dump releases or files that have been handled after pg_dump wrote them. gzip(1) accepts trailing zeros with a warning. A fix meant for back branches would likely be asked to keep the old acceptance or downgrade these cases to warnings.

**Question:** Has every pg_dump release that can write gzip directory or custom archives always written a non-empty gzip member, and why should trailing bytes be an error and not a warning?

<sub>fixable, level 2, backpatchability; messages [#3](https://www.postgresql.org/message-id/001b01dd5d3f%24c6a15e90%2453e41bb0%24%40gmx.net); `standards-compat-1`</sub>

## Questions for the author

1. Can the series be split into a minimal pg_restore-only fix for back branches and a master-only refactor of astreamer_gzip?
2. What does the fix look like for 14 and 15, which lack compress_gzip.c?
3. Has every pg_dump release that writes gzip archives always written at least one gzip member, so that no existing dump contains a zero-byte data file?
4. Why are trailing bytes after the last gzip member an error and not a warning, given that pg_restore accepts them today?
5. Does the refactored astreamer_gzip change behaviour or add translatable messages in pg_basebackup, pg_verifybackup or pg_waldump?

## Touch points

- **ABI / extension API:** The patch adds a shared frontend helper in src/include/fe_utils/gzip.h and changes astreamer_gzip to use it.
- **pg_dump / restore round-trip:** pg_restore now rejects zero-byte gzip data files and trailing bytes after the last gzip member.
- **backpatchability:** Message #3 asks for a back-patch to 14; 14 and 15 lack compress_gzip.c, and nls.mk changes touch four tools.
- **maintenance burden ("who maintains this in 10 years"):** Postgres takes over gzip stream handling from zlib's gzread(), and back branches may need two or three versions of the fix.

## Who would commit it, and when

- Owner: qualified committer engaged. Daniel Gustafsson said the patch is on his TODO list, but no committer review is visible yet. The owner needs to know pg_dump's compression code and the astreamer code.
- Priority: proceed; audience a sizeable minority; review cost medium. Message #3 reports a production failure, and a truncated restore that exited 0, with zlib 1.3.2, which the Windows binaries of 14 to 18 ship. The 35 KiB fix touches pg_dump and shared fe_utils code, and back branches need their own versions.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 4 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
