# Design pre-review: psql single-step mode woes

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7318](https://commitfest.postgresql.org/patch/7318/) (Needs review); authors: Heikki Linnakangas (heikki)
- Thread: [psql single-step mode woes](https://www.postgresql.org/message-id/flat/7f0278c4-8907-4db7-b1d7-31de89b41c44%40iki.fi), started 2026-07-04
- Messages read: 3 of 3
- Patch prose read: 0001-Improvements-to-psql-single-step-mode.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

No surviving objection questions the goal: the patch fixes defects in an existing psql mode, and two reviewers reproduced them. Two fixable points remain. standards-compat-2 asks which of the reviewer's fixes go to stable branches, since 0002 changes what EOF does at the prompt. standards-compat-1 asks what psql prints and returns with non-tty stdin. Neither would stop a committer from reading further.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, nobody can check that every error path restores the sigjmp_buf pointer, or what psql does with non-tty stdin. The reviewer's backpatch argument is cut off in message #3.

## The proposal, restated

**Problem.** psql single-step mode (SINGLESTEP, --single-step) misbehaves at its confirmation prompt. Ctrl-C does not leave the prompt, Ctrl-D runs all later queries (even in later \i scripts), any input other than "x" runs the query, and piped stdin answers its own prompts. The users are those who run scripts step by step to check each query.

**Proposed design.** The confirmation prompt reads a full line with gets_interactive() (tab completion off), asks again on unknown input, and exits at once on Ctrl-C. Ctrl-D cancels only the current script, and nothing runs in single-step mode when stdin is not a tty. A reviewer added fixes for master: a per-call sigjmp_buf with a global pointer, and a Ctrl-D fix that stops later backslash commands on the same line.

**User-visible surface.**
- Single-step prompt: input other than empty line or "x" now repeats the prompt instead of running the query.
- Single-step prompt: Ctrl-C cancels immediately; Ctrl-D cancels the current script without affecting later \i scripts.
- --single-step / SINGLESTEP with non-tty stdin no longer executes anything.
- Ctrl-D at the prompt no longer runs later backslash commands on the same line, and no longer exits psql without readline (reviewer's 0002).

**What the author claims.**
- The old behaviour surprises users, and most would expect Ctrl-C to leave the prompt.
- Running the query on unrecognized input is a dangerous default for a mode meant for careful checking.
- Single-step mode on non-tty stdin makes no sense; -f scripts still work.
- The reviewer claims nested MainLoop() reuse of sigint_interrupt_jmp crashes psql on Ctrl-C at \prompt, back to 8.3.

**Thread state.** Jelte Fennema-Nio liked the patch and asked for a distinct re-prompt naming the bad input. Rui Zhao found a Ctrl-C crash in the patch and Ctrl-D gaps, posted fixes 0001-0003, and began discussing backpatching; the message is truncated and Heikki has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-07-04; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Backpatching the Ctrl-D fix changes stable-branch behaviour at EOF

fixable · level 1b, rubric item 6 · backpatchability, psql UX conventions · `standards-compat-2`

A user on a stable branch drives single-step mode from a pipe or an expect script that ends with EOF, so all queries run. After a minor update with 0002, EOF cancels the script and skips later backslash commands. Their job now stops partway, and they learn this from missing results after the update.

Message #3 frames 0002 as a fix for master, then starts to discuss backpatching 0001 and 0002, and the message is cut off there. The SIGINT crash fix in 0001 is a clear bug fix. 0002 changes what EOF means at the prompt, and that behaviour has existed in every client back to at least 13. A committer would likely ask that only the crash fix go to stable branches.

Thread: [#3 (Rui Zhao, 2026-10-18)](https://www.postgresql.org/message-id/CAHWVJhF3_3V3v%2BYcrvB6rV3JRK_vGUpQKL3N5SFmZtQo%3D2nMbw%40mail.gmail.com)

**Question:** Which of 0001 and 0002 is proposed for stable branches, and why is the change in EOF behaviour acceptable in a minor release?

Counter-check: held. Message #3 starts to discuss backpatching 0001 and 0002 and is cut off, so the visible thread does not answer the question. 0002 changes what EOF does at the prompt on every client from 13, which a committer would likely ask about before a minor release.

### 2. Single-step with non-tty stdin silently changes from running to running nothing

fixable · level 2, rubric item 12 · design, psql UX conventions, backpatchability · `standards-compat-1`

A cron job runs `psql --single-step -f load.sql < /dev/null`, or pipes a script into psql with SINGLESTEP set in a shared psqlrc. Before the change, EOF at the prompt made every query run. After the upgrade nothing runs, and if psql exits 0 with no error, the operator finds out only when the data is missing.

Point 5 of the commit message turns an existing working, if odd, invocation into a no-op. The proposal does not say whether psql reports an error, prints a warning, or what exit status it returns. It also touches no documentation, so the --single-step entry would not describe the new rule. psql normally handles unusable option combinations by refusing loudly at startup, and a silent skip departs from that.

Thread: [#1 (Heikki Linnakangas, 2026-07-04)](https://www.postgresql.org/message-id/7f0278c4-8907-4db7-b1d7-31de89b41c44%40iki.fi), [#3 (Rui Zhao, 2026-10-18)](https://www.postgresql.org/message-id/CAHWVJhF3_3V3v%2BYcrvB6rV3JRK_vGUpQKL3N5SFmZtQo%3D2nMbw%40mail.gmail.com)

**Question:** With --single-step and non-tty stdin, what does psql print and what exit status does it return, and where is this documented?

Counter-check: held. The commit message says nothing runs with non-tty stdin, but not what psql prints or which exit status it returns, and the patch touches no documentation. The `-f` example is doubtful because the commit message says `-f` still works, but the piped case stands.

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
| psql UX conventions | **yes** | The patch changes the single-step prompt, Ctrl-C and Ctrl-D handling, and --single-step with non-tty stdin. |
| security / privileges | no |  |
| backpatchability | **yes** | Message #3 starts to discuss backpatching 0001 and 0002, and 0002 changes EOF behaviour on clients 13 through 19. |
| maintenance burden ("who maintains this in 10 years") | **yes** | Reviewer's 0001 replaces the global sigint_interrupt_jmp with local buffers and a global pointer in several psql functions. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which of the reviewer's 0001 and 0002 are proposed for stable branches, and why is the change in EOF behaviour at the prompt acceptable in a minor release?
2. With --single-step or SINGLESTEP and non-tty stdin, what does psql print and what exit status does it return?
3. Does the non-tty rule also apply when SINGLESTEP is set in a psqlrc file and a script is piped into psql?
4. Will the psql documentation describe the new re-prompt, the Ctrl-D behaviour and the non-tty rule?
5. Does the author accept the reviewer's split into 0001 and 0002 for master plus the feature patch 0003, and the distinct re-prompt text from message #2?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Heikki Linnakangas wrote the original patch. The owner needs to know psql input handling and its SIGINT longjmp code.
- **Liability:** The owner signs for psql's SIGINT and EOF handling at every interactive read, including the new per-call sigjmp_buf pointer, and for any stable-branch behaviour change if 0002 is backpatched.
- **Author follow-up:** The author has not replied in the visible thread since message #1, and the entry needs a rebase; the reviewer posted a rebased series in message #3.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** niche
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Single-step mode has few users, but the patch fixes unsafe behaviour and the reviewer's 0001 fixes a Ctrl-C crash that reaches all psql users of nested scripts with \prompt. The patch stays within psql, but signal handling with sigsetjmp needs careful review.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-1` New sigjmp_buf pointer convention must be kept by every later psql prompt. Refuted (no failure scenario): The scenario already exists on master in a wider form: any nested sigsetjmp on the shared global buffer leaves a stale context. 0001 narrows it and follows the PG_TRY pattern, and asking for a macro is a code-structure point, which is out of scope.

### Review angles

- prior-attempts: 0 objection(s). No precedent found. The only listed row, [P13], is about stopping psql command execution after a connection reset, with no recorded reason; it shares only the psql area. Message #1 points to an earlier thread about PQcancel, but only as the place where the single-step complaint was first mentioned, not as a prior attempt at this fix.
- standards-compat: 2 objection(s). The interactive prompt changes (re-prompt on unknown input, Ctrl-C exits at once) add no syntax, GUC or dump surface, and I raise no objection to them.
- replication-ondisk: 0 objection(s). The proposal changes only psql client-side input handling: the single-step prompt, SIGINT jump buffers and EOF handling. It touches no on-disk format, WAL, catalog, wire protocol or server ABI. The gets_interactive() signature change is internal to psql and does not affect extensions.
- security: 0 objection(s). The proposal changes only client-side psql behaviour at the single-step prompt. It adds no privilege, role, server code path or monitoring exposure, and it gives no user access to data or sessions they lack today. Each change makes the default safer: unknown input no longer runs the query, Ctrl-D no longer runs later commands, and non-tty stdin runs nothing. The Ctrl-C crash in message #3 is a local client abort with no cross-user effect, so it belongs to the correctness angle. Without the code, I could not check whether the sigjmp_buf pointer changes leave a stale pointer on any error path.
- maintenance-fit: 1 objection(s). From the maintenance angle the proposal fits core: psql behaviour, no hot path, no new dependency, and a committer wrote it, with a reviewer who split it into fixes for master first. The one concern is the hand-kept sigjmp_buf pointer convention.
- already-exists: 0 objection(s). This is a bug fix to psql's existing single-step mode, and it adds no new mechanism. No core feature, option, extension or external tool gives a working Ctrl-C, Ctrl-D or strict input handling at that prompt, so this angle raises nothing. The listed precedent [P13] covers connection reset, an unrelated topic.
- demand: 0 objection(s). No demand objection. The patch fixes defects in an existing documented mode: Ctrl-D runs later scripts unconfirmed, and nested Ctrl-C crashes psql in every client since 8.3, so it is not new surface needing users. Two other people tested it and reproduced the problems (messages #2, #3), and the cost falls on single-step users only.

### Precedents retrieved

- P1: [session username in default psql prompt](https://commitfest.postgresql.org/patch/4856/) (cf, Withdrawn, 2024-03-23) - judged not relevant; reason summarised from its thread
- P2: [\[PATCH v2\] Time option added to psql prompt (%T)](https://commitfest.postgresql.org/patch/4227/) (cf, Returned with feedback, 2023-07-03) - judged not relevant; reason summarised from its thread
- P3: [Add new protocol message to make GUC_REPORT configurable and use it for psql prompt substitution of %N with the current user](https://commitfest.postgresql.org/patch/4166/) (cf, Withdrawn, 2024-01-30) - judged not relevant; reason summarised from its thread
- P4: [Add %r substitution for psql prompts to show recovery status](https://commitfest.postgresql.org/patch/1402/) (cf, Returned with feedback, 2018-01-22) - judged not relevant; reason summarised from its thread
- P5: [New psql prompt substitution %r (m = master, r = replica)](https://commitfest.postgresql.org/patch/706/) (cf, Returned with feedback, 2016-08-12) - judged not relevant; reason summarised from its thread
- P6: [Refactoring of connection with password prompt loop for frontends](https://commitfest.postgresql.org/patch/2248/) (cf, Returned with feedback, 2019-09-27) - judged not relevant; reason summarised from its thread
- P7: [Restart pg_usleep when interrupted](https://commitfest.postgresql.org/patch/5161/) (cf, Withdrawn, 2024-09-04) - judged not relevant; reason summarised from its thread
- P8: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant; reason summarised from its thread
- P9: [Don't ignore recovery conflict interrupt on blocked writing](https://commitfest.postgresql.org/patch/5504/) (cf, Withdrawn, 2025-03-07) - judged not relevant
- P10: [Strip Windows newlines from extension script files manually.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=6cfb3a33746990602c790199adc2debb2c4bbb87) (revert, Partially reverted, 2024-10-28) - judged not relevant
- P11: [psql: Allow editing query results with \gedit](https://commitfest.postgresql.org/patch/4788/) (cf, Rejected, 2024-05-14) - judged not relevant
- P12: [Tab-completion refactoring and word-removal feature (was IF-NOT_EXISTS for tab-completion)](https://commitfest.postgresql.org/patch/855/) (cf, Returned with feedback, 2017-03-17) - judged not relevant
- P13: [Stop psql interactive command execution upon connection reset](https://commitfest.postgresql.org/patch/830/) (cf, Returned with feedback, 2016-12-02) - used in the review
- P14: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P15: [psql: Add tab-complete for optional view parameters](https://commitfest.postgresql.org/patch/4053/) (cf, Returned with feedback, 2023-07-06) - judged not relevant
- P16: [tab-completion debug print](https://commitfest.postgresql.org/patch/1917/) (cf, Returned with feedback, 2019-02-02) - judged not relevant
- P17: [document that psql's FETCH_COUNT does not work with combined (\;) queries](https://commitfest.postgresql.org/patch/2219/) (cf, Returned with feedback, 2020-09-30) - judged not relevant
- P18: [Resume vacuum and autovacuum from interruption and cancellation](https://commitfest.postgresql.org/patch/2211/) (cf, Returned with feedback, 2020-04-08) - judged not relevant
- P19: [Allow an extention to be updated without a script](https://commitfest.postgresql.org/patch/4154/) (cf, Withdrawn, 2023-02-01) - judged not relevant
- P20: [Add extra statistics to explain for Nested Loop](https://commitfest.postgresql.org/patch/2765/) (cf, Returned with feedback, 2022-10-12) - judged not relevant
- P21: [improve tab-completion behavior when parse text in single quote](https://commitfest.postgresql.org/patch/3267/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P22: [prefetching for nestloop / indexscan](https://commitfest.postgresql.org/patch/5208/) (cf, Withdrawn, 2024-11-22) - judged not relevant
- P23: [Revert "psql: Show all query results by default"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=fae65629cec824738ee11bf60f757239906d64fa) (revert, Reverted, 2021-04-15) - judged not relevant
- P24: [possibility to specify sort columns for psql describe commands in verbose mode](https://commitfest.postgresql.org/patch/1032/) (cf, Rejected, 2018-01-12) - judged not relevant

### Warnings

- synthesis said NEEDS-COMMITTER-EYES but listed a level 1b objection (standards-compat-2); changed to NEEDS-DESIGN-DISCUSSION

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 98 s; cost reported by the CLI: $1.12
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 1 of 24 kept
- Tone check: 61 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:40:08.340Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
