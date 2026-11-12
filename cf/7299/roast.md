# Design pre-review: Avoid emitting core dumps when not needed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 7299](https://commitfest.postgresql.org/patch/7299/); [thread](https://www.postgresql.org/message-id/flat/20231118222911.trznxgya4hvzapva%40awork3.anarazel.de), 13 of 13 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at levels 1a or 1b: message #1 states the problem and messages #2 to #5 support the direction. Two fixable level 2 points remain. maintenance-fit-1 asks why the patch matches ENOSPC at each call site when one rule on the SQLSTATE would cover EDQUOT and future sites. standards-compat-1 notes that the postmaster log line and exit status change for a marked PANIC. Both need a committer who knows error handling and the postmaster.

Not checked: Without the code, this review could not check that _exit(2) and abort() differ only in the core file, that the nested PANIC precedence holds, or that the postmaster control-file path sees the GUC.

## Prior attempts

None found before 2023-11-18.

## Objections a reviewer is likely to raise

### 1. Per-call-site errno marking instead of one rule keyed on the SQLSTATE

An operator runs pg_wal on a quota-limited volume. A WAL write fails with EDQUOT inside a critical section and is promoted to PANIC. Each crash-restart cycle still writes a core, because only ENOSPC is matched. The same happens at a WAL write site added in a later release that nobody marked.

v2-0002 lists the marked paths by hand, and v5 added walreceiver only because a reviewer noticed it (message #12, message #13). Nothing fails when a new write path is left unmarked. errcode_for_file_access() already maps both ENOSPC and EDQUOT to ERRCODE_DISK_FULL, so one check in errfinish() on that SQLSTATE would cover every current and future site. Message #9 suggested deciding from the error code, and the thread did not weigh this option against errno matching.

**Question:** Why is matching ENOSPC at each call site better than one errfinish() rule that skips the core for a PANIC with SQLSTATE 53100?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#9](https://www.postgresql.org/message-id/CAN55FZ3YH2%3DFtr%3DQ---LKXxF_m%2BwcgHnFTNw%3DJ26hEn2i%2BrLWg%40mail.gmail.com), [#12](https://www.postgresql.org/message-id/CAN55FZ3F1eS7i2UQE3ptFuLOe0BTK6reXgF9rRRpNEDh4zDhqw%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAJTYsWV%3DKjO5h_Pn5YGqgeoe%2B77wawQ1u00WmLOjHqcF7QdODQ%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. Marked PANIC changes the postmaster crash log line and exit status

An operator alerts on the postmaster line "terminated by signal 6: Aborted", or on SIGABRT exits seen by a process supervisor. After upgrade, a WAL ENOSPC PANIC shows up as "exited with exit code 2" instead. Alerts that matched the signal stop firing, and the operator finds out only during the next disk-full outage.

The brief already lists SIGABRT-based monitoring as an open question, and no message in the thread takes it up. Children killed during crash handling also leave with exit code 2, so the new exit status looks like an ordinary secondary exit. In message #6 Andres wanted core-dumping errors to be easy to search for in logs, and this design makes the culprit harder to find. v5 has no reply yet, so nobody has weighed the change in what users see.

**Question:** What does the postmaster log for a marked PANIC, and how does an operator tell it apart from the other children's exit-code-2 exits after a crash?

<sub>fixable, level 2, design; messages [#1](https://www.postgresql.org/message-id/20231118222911.trznxgya4hvzapva%40awork3.anarazel.de), [#6](https://www.postgresql.org/message-id/20231120233518.l7p5vzlqaxwkztlh%40awork3.anarazel.de), [#7](https://www.postgresql.org/message-id/CAJTYsWV601xS0D_SrZ111VX0N%2BtRe2fjK-VjY75v1gNJ93%2BjkQ%40mail.gmail.com), [#13](https://www.postgresql.org/message-id/CAJTYsWV%3DKjO5h_Pn5YGqgeoe%2B77wawQ1u00WmLOjHqcF7QdODQ%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Why is matching ENOSPC at each call site better than one errfinish() rule that skips the core for a PANIC with SQLSTATE 53100?
2. Should EDQUOT and EROFS also skip the core dump, and if not, why is ENOSPC different?
3. What does the postmaster log for a marked PANIC, and how does an operator tell it apart from other children that exit with code 2?
4. Does a marked PANIC still produce core files from the other backends when send_abort_for_crash is on?
5. Are extensions expected to call errabort() and errnocoredump_on_errno(), and is that API meant to be stable?

## Touch points

- **WAL / replication / standby:** The patch marks ENOSPC errors in WAL write, sync and recovery read paths in xlog.c and xlogrecovery.c, and in walreceiver.
- **ABI / extension API:** ErrorData gets a no_core_dump field, and elog.h gains errabort() and errnocoredump_on_errno().
- **new GUC:** The patch adds the developer option force_core_dump_on_panic.
- **maintenance burden ("who maintains this in 10 years"):** Each PANIC-capable write path must be marked by hand, and nothing fails when a new path is left unmarked.

## Who would commit it, and when

- Owner: silent. A committer who knows elog.c error handling and the postmaster crash-restart path would have to own it. Committers shaped the idea in 2023, but none has replied to v1 through v5.
- Priority: needs champion; audience a sizeable minority; review cost medium. Operators who collect core dumps in production benefit, and message #1 reports the problem at fleet scale. The patch is small and its cost is limited to PANIC paths, but it touches elog.c and WAL write paths, and no committer has replied to any version.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 0 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
