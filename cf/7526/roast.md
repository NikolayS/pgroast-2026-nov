# Design pre-review: Print measurements on CTRL-C in pgbench

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7526](https://commitfest.postgresql.org/patch/7526/); [thread](https://www.postgresql.org/message-id/flat/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection is fatal, and the idea fits pgbench as a core tool. Three fixable points remain: already-exists-1 asks why `-T` with `-P` falls short, standards-compat-1 shows that exit(130) lets a calling shell loop continue, and maintenance-fit-2 notes that Windows behaviour is undefined. The author calls the patch a POC and leaves the exit code open in message #1, so these are open design points and do not count against the goal.

Not checked: Without the code, this review could not check how v2 handles in-flight transactions in the statistics, whether the handler is async-signal-safe, or how the one-second wait cap interacts with `-R` and `\sleep`.

## Prior attempts

None found before 2026-10-26.

## Objections a reviewer is likely to raise

### 1. Existing -T and -P options already cover short manual runs

A developer who wants a 5-10 second comparison can run `pgbench -T 10 -P 1` and get the full summary plus per-second TPS and latency. The patch adds a new SIGINT path, a new exit status and a one-second wait cap that every pgbench run then carries. The open Windows question has to be resolved and maintained for a case these options already handle.

Message #1 describes runs of a known short length ("5-10 seconds"). That is the use case for `-T`, which ends with the normal summary, and `-P` prints TPS and latency while the run is going, so a user who presses Ctrl-C has already seen the numbers. The proposal does not mention either option or say why they fall short. Without that, the gain looks like convenience: no need to choose the duration in advance. A committer would likely ask whether that convenience is worth the permanent signal-handling code.

**Question:** What does a Ctrl-C summary give that `-T` with `-P 1` does not, for the short comparison runs described in message #1?

<sub>fixable, level 1b, doesn't fit what Postgres is about (extension / external tool / already solved); messages [#1](https://www.postgresql.org/message-id/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com); `already-exists-1`</sub>

### 2. Exit 130 instead of re-raising SIGINT breaks shell loop interruption

A user runs pgbench inside a bash loop or a Makefile, for example for each scale factor, and presses Ctrl-C. Bash sees pgbench exit normally with status 130 rather than die from SIGINT, so it keeps going and starts the next pgbench run. The user has to press Ctrl-C again for every remaining iteration.

Today pgbench dies from SIGINT, and calling shells rely on that to stop the whole script. The usual convention for programs that catch SIGINT is to clean up, reset the handler and re-raise the signal, not to exit(130). The author says in message #1 that the exit code is still open. The patch adds no documentation hunk, so the documented exit statuses, 0, 1 and 2 with partial results, would gain an undocumented fourth value.

**Question:** After printing the summary, will pgbench re-raise SIGINT so calling shells see a signal death, and how will the exit-status documentation change?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/F9B91838-37D4-420A-8DC4-5ADA4861B880%40yandex-team.ru), [#2](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/39384C28-7D89-4E8A-A85F-952309075357%40yandex-team.ru); `standards-compat-1`</sub>

### 3. Windows behaviour left undefined for a cross-platform tool

A Windows user presses Ctrl-C during a run. pgbench may print nothing, print a summary while threads keep running, or hang until a wait expires. The user finds out only on that platform, and the bug report then goes to a committer who has no Windows setup.

Windows delivers console Ctrl-C through a separate handler thread rather than POSIX SIGINT, so a design built on signal state and EINTR does not carry over unchanged. Message #2 points to an earlier thread on exactly this issue, and message #3 only promises to read it. The touched files list only pgbench.c, with no tests and no documentation for either platform.

**Question:** What does a first and a second Ctrl-C do on Windows with v2, and how does a test show that it matches the Unix behaviour?

<sub>fixable, level 2, design; messages [#2](https://www.postgresql.org/message-id/CALdSSPhmwR4__7QpY2fU5knOd41cGPb9RtsL%2BKedVHyBGMiuxA%40mail.gmail.com), [#3](https://www.postgresql.org/message-id/39384C28-7D89-4E8A-A85F-952309075357%40yandex-team.ru); `maintenance-fit-2`</sub>

## Questions for the author

1. What does a Ctrl-C summary give that `-T` with `-P 1` does not, for the short comparison runs described in message #1?
2. After printing the summary, will pgbench reset the handler and re-raise SIGINT so a calling shell loop or Makefile stops, or exit with status 130?
3. What do a first and a second Ctrl-C do on Windows with v2?
4. How do the discarded in-flight transactions affect the TPS, latency and per-script numbers in the summary?
5. How will the pgbench documentation on exit status and interrupted runs change?

## Touch points

- **backpatchability:** see objection standards-compat-1: Exit 130 instead of re-raising SIGINT breaks shell loop interruption
- **maintenance burden ("who maintains this in 10 years"):** The patch adds a SIGINT path, a new exit status and a one-second wait cap in pgbench.c, with Windows handling still open.
- **doesn't fit what Postgres is about (extension / external tool / already solved):** `pgbench -T 10 -P 1` already gives a summary and per-second numbers for the short runs in message #1.

## Who would commit it, and when

- Owner: silent. Someone who knows pgbench's thread and wait loop and frontend signal handling, including Windows console Ctrl-C handling.
- Priority: fix design first; audience a sizeable minority; review cost low. The patch is about 4.6 KB in one file and serves developers who run pgbench by hand, though the thread shows only one such workflow. The exit behaviour and the Windows behaviour need an answer before code review is useful.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 2 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
