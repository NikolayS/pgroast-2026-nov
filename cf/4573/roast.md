# Design pre-review: pg_ctl start may return 0 even if the postmaster has been already started on Windows

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 4573](https://commitfest.postgresql.org/patch/4573/); [thread](https://www.postgresql.org/message-id/flat/TYAPR01MB586654E2D74B838021BE77CAF5EEA%40TYAPR01MB5866.jpnprd01.prod.outlook.com), 43 of 43 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: the bug is stated in message #1 and committers agree it should be fixed. The surviving objections are fixable level 2 points about the CreateProcess() design. prior-attempts-1 is the strongest, because message #34 reverted /D for the same user-visible effect. maintenance-fit-1 needs a committer who knows Windows token handling to sign.

Not checked: Without the code, nobody can tell whether v6 shares token code with CreateRestrictedProcess(), keeps the 2-second wait, or cleans up the two global PID variables. The Unicode executable check failures that Harrison reports are also unexplained.

## Prior attempts

None found before 2023-09-07.

## Objections a reviewer is likely to raise

### 1. Dropping cmd.exe brings back the AutoRun change reverted for v17

A Windows administrator sets a cmd.exe AutoRun entry, for example `chcp` or environment variables, that the postmaster inherits today through the launcher shell. After the CreateProcess() change, `pg_ctl start` runs postgres.exe directly, so the entry never runs. The administrator finds out only when server log encoding or environment differs after an upgrade.

Message #32 argued that it is not Postgres's job to second-guess the user's AutoRun settings, and that /D on only some %comspec% callers is inconsistent. Message #34 reverted /D for that reason. Running postgres.exe without cmd.exe has the same user-visible effect for pg_ctl start, while pgwin32_system() keeps running AutoRun. Messages #40-#43 do not discuss this.

**Question:** Why is skipping AutoRun acceptable for pg_ctl start under CreateProcess() when the same effect from /D was reverted, and what happens to other %comspec% callers?

<sub>fixable, level 2, prior rejection; messages [#11](https://www.postgresql.org/message-id/20230922.161551.320043332510268554.horikyota.ntt%40gmail.com), [#32](https://www.postgresql.org/message-id/20240630021211.f3.nmisch%40google.com), [#33](https://www.postgresql.org/message-id/ZoeCemTDhF0OxuB2%40paquier.xyz), [#34](https://www.postgresql.org/message-id/Zos5ojel68kBT79X%40paquier.xyz), [#40](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#42](https://www.postgresql.org/message-id/202602021323.y4ctu6t5biso%40alvherre.pgsql); `prior-attempts-1`</sub>

### 2. Second copy of the privilege-dropping logic for the postmaster launch

A later patch changes which groups or privileges CreateRestrictedProcess() removes, for example after a Windows change. The new CreateProcessAsUser() path in start_postmaster() keeps the old token rules. An elevated pg_ctl then starts a postmaster with more rights than intended, and nothing in CI reports the difference.

Message #40 says the new code builds its own restricted token, "matching the behavior of the existing CreateRestrictedProcess() function". Matching by copy means two places must stay in sync, and only elevated-privilege tests would show a gap. Message #43 adds a refinement that opens log files under the restricted identity, so this security-sensitive code grows further. The prose does not say whether v6 shares one helper with CreateRestrictedProcess().

**Question:** Does the direct launch reuse the same token-building code as CreateRestrictedProcess(), so one change covers both paths?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#40](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#43](https://www.postgresql.org/message-id/CAAH-eSX%3DM5fsShzL401VB7t82qiAsv2difOHnPd0qWA%2BHBMc-w%40mail.gmail.com); `maintenance-fit-1`</sub>

### 3. pg_ctl -o and -l lose cmd.exe parsing without a stated contract

A Windows service is registered with pg_ctl register -o containing %PGDATA2% or caret-escaped quotes, and the stored command line is expanded by cmd.exe at each start. After the change, CreateProcess() passes the string literally and the postmaster rejects the option or reads the wrong path. The user finds out at the next service restart.

Today start_postmaster() builds a cmd.exe command line, so -o text and the -l redirection go through cmd.exe quoting, %VAR% expansion and append semantics. Message #40 describes handle-based redirection, but not whether -o text is parsed the same way or whether the log file is still appended to. Message #43 tests paths with spaces and U+00E9, but not shell metacharacters or variables in -o.

**Question:** Is the postmaster command line, including -o text with %VAR%, ^ and quotes, and the -l append behaviour, identical with and without cmd.exe, and is any difference documented?

<sub>fixable, level 2, design; messages [#40](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com), [#43](https://www.postgresql.org/message-id/CAAH-eSX%3DM5fsShzL401VB7t82qiAsv2difOHnPd0qWA%2BHBMc-w%40mail.gmail.com); `standards-compat-2`</sub>

### 4. Fixed 2-second wait to detect postgres -C

On a loaded Windows CI machine, a postgres -C child or an early-failing postmaster takes longer than 2 seconds to exit. pg_ctl takes the wrong branch and the test fails only sometimes. Developers then find intermittent failures that look like unrelated Windows noise.

Message #40 says the direct launch waits 2 seconds when there is no log file, to tell postgres -C apart from server startup. That puts a new timing threshold into code whose main fault is already a 2-second clock allowance (message #1). Timing thresholds tend to need retuning, and the material does not say whether v6 keeps this wait or replaces it with an exit-status check.

**Question:** Can the launch tell postgres -C from server startup without a fixed delay, and does v6 still contain the 2-second wait?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/TYAPR01MB586654E2D74B838021BE77CAF5EEA%40TYAPR01MB5866.jpnprd01.prod.outlook.com), [#40](https://www.postgresql.org/message-id/e4025275-0f97-4a3e-b107-a85e60ccf0f7%40gmail.com); `maintenance-fit-3`</sub>

## Questions for the author

1. Why is skipping cmd.exe AutoRun acceptable under CreateProcess() when message #34 reverted /D for the same effect, and what happens to other %comspec% callers?
2. Does the direct launch share the token-building code with CreateRestrictedProcess(), so one change covers both paths?
3. Are -o text with %VAR%, ^ and quotes, and the -l append behaviour, the same with and without cmd.exe?
4. Does v6 still wait 2 seconds to detect postgres -C, and can the launch tell the two cases apart without a fixed delay?
5. Which branches does the fix target, and does the bug exist in v16 as message #36 questions?

## Touch points

- **on-disk format / pg_upgrade:** The token design would add a value to postmaster.pid, and pg_upgrade relies on the pg_ctl start exit code.
- **security / privileges:** The CreateProcess() design builds its own restricted token to drop privileges for the postmaster.
- **backpatchability:** It is a bug fix, the /D change was reverted for v17, and message #42 suggests a smaller fix for back branches.
- **maintenance burden ("who maintains this in 10 years"):** About 250 lines of Windows-only launch and token code in pg_ctl.c that few reviewers can check.

## Who would commit it, and when

- Owner: qualified committer engaged. Álvaro Herrera states a design preference in message #42; the owner needs to know Windows process creation and restricted tokens, and message #38 notes that reviewers are not Windows experts.
- Priority: needs champion; audience niche; review cost medium. The bug affects Windows scripts and tools that trust the pg_ctl start exit code, with CI flakiness as the only shown impact. The entry has moved through 13 CommitFests without a design decision, and v6 needs a committer who knows Windows to review it.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 3 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
