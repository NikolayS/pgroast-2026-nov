# Design pre-review: Let's drop support for named POSIX semaphores

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7513](https://commitfest.postgresql.org/patch/7513/); [thread](https://www.postgresql.org/message-id/flat/507767.1792981204%40sss.pgh.pa.us), 11 of 11 messages read; no code read.

## Verdict

**NO-FUNDAMENTAL-OBJECTION** (confidence: medium). No objection survived at levels 1a, 1b or 2. The proposal states the problem and the change, removes an untested path that meson cannot select, and leaves SysV and unnamed POSIX semaphores in place. The one open point is whether an out-of-tree configure build still selects named semaphores; the material shows none.

Not checked: The code was not read, so this review cannot confirm that the configure and darwin template changes leave SysV and unnamed POSIX selection unchanged on every platform. Only 3 of 11 messages were shown.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

No objection held up in the counter-check.

## Questions for the author

1. Does any packager or out-of-tree build still select named POSIX semaphores through configure, for example by setting USE_NAMED_POSIX_SEMAPHORES?
2. Will the release notes mention the removal so that anyone who builds with named semaphores learns of it?
3. Will a v2 include the cleanup from message #2: the unused IPCProtection macro, the unneeded includes, and the runtime.sgml sentence?

## Touch points

- **ABI / extension API:** The patch removes USE_NAMED_POSIX_SEMAPHORES from pg_config.h.in, a macro that out-of-tree code could test.
- **maintenance burden ("who maintains this in 10 years"):** The patch deletes a semaphore path in posix_sema.c and its configure, meson and darwin template logic, which lowers the burden.

## Who would commit it, and when

- Owner: qualified committer engaged. Tom Lane, a committer, wrote the patch, and Heikki Linnakangas supported it in message #3. The owner needs to know the port-level semaphore code and the configure and meson build logic.
- Priority: proceed; audience niche; review cost low. The patch is small, touches six files, and only removes code that nobody tests and meson cannot select. A committer wrote it, three people support it, and the one concern raised in the thread was withdrawn.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
