# Design pre-review: psql single-step mode woes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7318](https://commitfest.postgresql.org/patch/7318/); [thread](https://www.postgresql.org/message-id/flat/7f0278c4-8907-4db7-b1d7-31de89b41c44%40iki.fi), 3 of 3 messages read; no code read.

## Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium). No surviving objection questions the goal: the patch fixes defects in an existing psql mode, and two reviewers reproduced them. Two fixable points remain. standards-compat-2 asks which of the reviewer's fixes go to stable branches, since 0002 changes what EOF does at the prompt. standards-compat-1 asks what psql prints and returns with non-tty stdin. Neither would stop a committer from reading further.

Not checked: Without the code, nobody can check that every error path restores the sigjmp_buf pointer, or what psql does with non-tty stdin. The reviewer's backpatch argument is cut off in message #3.

## Prior attempts

None found before 2026-07-04.

## Objections a reviewer is likely to raise

### 1. Backpatching the Ctrl-D fix changes stable-branch behaviour at EOF

A user on a stable branch drives single-step mode from a pipe or an expect script that ends with EOF, so all queries run. After a minor update with 0002, EOF cancels the script and skips later backslash commands. Their job now stops partway, and they learn this from missing results after the update.

Message #3 frames 0002 as a fix for master, then starts to discuss backpatching 0001 and 0002, and the message is cut off there. The SIGINT crash fix in 0001 is a clear bug fix. 0002 changes what EOF means at the prompt, and that behaviour has existed in every client back to at least 13. A committer would likely ask that only the crash fix go to stable branches.

**Question:** Which of 0001 and 0002 is proposed for stable branches, and why is the change in EOF behaviour acceptable in a minor release?

<sub>fixable, level 1b, backpatchability; messages [#3](https://www.postgresql.org/message-id/CAHWVJhF3_3V3v%2BYcrvB6rV3JRK_vGUpQKL3N5SFmZtQo%3D2nMbw%40mail.gmail.com); `standards-compat-2`</sub>

### 2. Single-step with non-tty stdin silently changes from running to running nothing

A cron job runs `psql --single-step -f load.sql < /dev/null`, or pipes a script into psql with SINGLESTEP set in a shared psqlrc. Before the change, EOF at the prompt made every query run. After the upgrade nothing runs, and if psql exits 0 with no error, the operator finds out only when the data is missing.

Point 5 of the commit message turns an existing working, if odd, invocation into a no-op. The proposal does not say whether psql reports an error, prints a warning, or what exit status it returns. It also touches no documentation, so the --single-step entry would not describe the new rule. psql normally handles unusable option combinations by refusing loudly at startup, and a silent skip departs from that.

**Question:** With --single-step and non-tty stdin, what does psql print and what exit status does it return, and where is this documented?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/7f0278c4-8907-4db7-b1d7-31de89b41c44%40iki.fi), [#3](https://www.postgresql.org/message-id/CAHWVJhF3_3V3v%2BYcrvB6rV3JRK_vGUpQKL3N5SFmZtQo%3D2nMbw%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which of the reviewer's 0001 and 0002 are proposed for stable branches, and why is the change in EOF behaviour at the prompt acceptable in a minor release?
2. With --single-step or SINGLESTEP and non-tty stdin, what does psql print and what exit status does it return?
3. Does the non-tty rule also apply when SINGLESTEP is set in a psqlrc file and a script is piped into psql?
4. Will the psql documentation describe the new re-prompt, the Ctrl-D behaviour and the non-tty rule?
5. Does the author accept the reviewer's split into 0001 and 0002 for master plus the feature patch 0003, and the distinct re-prompt text from message #2?

## Touch points

- **psql UX conventions:** The patch changes the single-step prompt, Ctrl-C and Ctrl-D handling, and --single-step with non-tty stdin.
- **backpatchability:** Message #3 starts to discuss backpatching 0001 and 0002, and 0002 changes EOF behaviour on clients 13 through 19.
- **maintenance burden ("who maintains this in 10 years"):** Reviewer's 0001 replaces the global sigint_interrupt_jmp with local buffers and a global pointer in several psql functions.

## Who would commit it, and when

- Owner: qualified committer engaged. Heikki Linnakangas wrote the original patch. The owner needs to know psql input handling and its SIGINT longjmp code.
- Priority: proceed; audience niche; review cost medium. Single-step mode has few users, but the patch fixes unsafe behaviour and the reviewer's 0001 fixes a Ctrl-C crash that reaches all psql users of nested scripts with \prompt. The patch stays within psql, but signal handling with sigsetjmp needs careful review.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details. Warnings in this run: 1.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
