# Design pre-review: Avoid emitting core dumps when not needed

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7299](https://commitfest.postgresql.org/patch/7299/) (Needs review); authors: Ayush Tiwari (aytiwari)
- Thread: [PANIC serves too many masters](https://www.postgresql.org/message-id/flat/20231118222911.trznxgya4hvzapva%40awork3.anarazel.de), started 2023-11-18
- Messages read: 13 of 13
- Patch prose read: v1-0001-Allow-selected-PANIC-errors-to-skip-core-dumps.patch, v1-0002-Avoid-core-dumps-for-WAL-disk-full-failures.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at levels 1a or 1b: message #1 states the problem and messages #2 to #5 support the direction. Two fixable level 2 points remain. maintenance-fit-1 asks why the patch matches ENOSPC at each call site when one rule on the SQLSTATE would cover EDQUOT and future sites. standards-compat-1 notes that the postmaster log line and exit status change for a marked PANIC. Both need a committer who knows error handling and the postmaster.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that _exit(2) and abort() differ only in the core file, that the nested PANIC precedence holds, or that the postmaster control-file path sees the GUC.

## The proposal, restated

**Problem.** Every PANIC calls abort(), so operational failures such as ENOSPC during WAL writes produce core files with little diagnostic value. Repeated crash-restart cycles can create thousands of identical cores, which fills disks further and hides real bugs in fleet-wide crash analysis. The audience is operators and developers who collect core dumps in production.

**Proposed design.** PANIC keeps its level and restart behavior, and ErrorData gets a no_core_dump flag. A marked PANIC exits with _exit(2) instead of abort(), so the postmaster still runs crash handling; a core-worthy outer PANIC wins over a nested marked error. Selected WAL, control-file and walreceiver write paths mark ENOSPC this way.

**User-visible surface.**
- New developer GUC force_core_dump_on_panic (boolean, postgresql.conf or command line) that restores abort() for marked PANICs.
- New ereport() auxiliary function errabort(bool) in the C API.
- New ereport() auxiliary function errnocoredump_on_errno(int), which marks the error when the saved errno matches.
- PANIC on ENOSPC in WAL and control-file I/O no longer produces a core file by default.
- New injection_points test function, renamed in v5 to injection_points_test_panic_core_dump.

**What the author claims.**
- A core file from a disk-full PANIC has little diagnostic value and can make disk pressure worse.
- Existing postmaster crash handling is preserved because exit code 2 is treated as a crash.
- Other errno values, corruption failures and internal-invariant PANICs still call abort().
- The postmaster reads postgresql.conf before its first control-file read, so the override GUC applies there.

**Thread state.** Andres Freund, Jeff Davis and Tom Lane discussed the idea in 2023 and favored keeping PANIC while changing call sites gradually. Nazir Bilal Yavuz reviewed v2 through v4; v5 answers his comments, and nobody has replied to it yet.

## 2. Prior attempts

No precedent found (corpus searched for items before 2023-11-18; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Per-call-site errno marking instead of one rule keyed on the SQLSTATE

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years") · `maintenance-fit-1`

An operator runs pg_wal on a quota-limited volume. A WAL write fails with EDQUOT inside a critical section and is promoted to PANIC. Each crash-restart cycle still writes a core, because only ENOSPC is matched. The same happens at a WAL write site added in a later release that nobody marked.

v2-0002 lists the marked paths by hand, and v5 added walreceiver only because a reviewer noticed it (message #12, message #13). Nothing fails when a new write path is left unmarked. errcode_for_file_access() already maps both ENOSPC and EDQUOT to ERRCODE_DISK_FULL, so one check in errfinish() on that SQLSTATE would cover every current and future site. Message #9 suggested deciding from the error code, and the thread did not weigh this option against errno matching.

Thread: [#9 (Nazir Bilal Yavuz, 2026-10-02)](https://www.postgresql.org/message-id/CAN55FZ3YH2%3DFtr%3DQ---LKXxF_m%2BwcgHnFTNw%3DJ26hEn2i%2BrLWg%40mail.gmail.com), [#12 (Nazir Bilal Yavuz, 2026-10-12)](https://www.postgresql.org/message-id/CAN55FZ3F1eS7i2UQE3ptFuLOe0BTK6reXgF9rRRpNEDh4zDhqw%40mail.gmail.com), [#13 (Ayush Tiwari, 2026-10-12)](https://www.postgresql.org/message-id/CAJTYsWV%3DKjO5h_Pn5YGqgeoe%2B77wawQ1u00WmLOjHqcF7QdODQ%40mail.gmail.com)

**Question:** Why is matching ENOSPC at each call site better than one errfinish() rule that skips the core for a PANIC with SQLSTATE 53100?

Counter-check: held. Marking call sites one by one follows what Tom Lane and Jeff Davis favored in messages #3 and #4, which lowers the weight. But the EDQUOT case is concrete, and the thread did not weigh the error-code rule raised in message #9.

### 2. Marked PANIC changes the postmaster crash log line and exit status

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

An operator alerts on the postmaster line "terminated by signal 6: Aborted", or on SIGABRT exits seen by a process supervisor. After upgrade, a WAL ENOSPC PANIC shows up as "exited with exit code 2" instead. Alerts that matched the signal stop firing, and the operator finds out only during the next disk-full outage.

The brief already lists SIGABRT-based monitoring as an open question, and no message in the thread takes it up. Children killed during crash handling also leave with exit code 2, so the new exit status looks like an ordinary secondary exit. In message #6 Andres wanted core-dumping errors to be easy to search for in logs, and this design makes the culprit harder to find. v5 has no reply yet, so nobody has weighed the change in what users see.

Thread: [#1 (Andres Freund, 2023-11-18)](https://www.postgresql.org/message-id/20231118222911.trznxgya4hvzapva%40awork3.anarazel.de), [#6 (Andres Freund, 2023-11-20)](https://www.postgresql.org/message-id/20231120233518.l7p5vzlqaxwkztlh%40awork3.anarazel.de), [#7 (Ayush Tiwari, 2026-09-12)](https://www.postgresql.org/message-id/CAJTYsWV601xS0D_SrZ111VX0N%2BtRe2fjK-VjY75v1gNJ93%2BjkQ%40mail.gmail.com), [#13 (Ayush Tiwari, 2026-10-12)](https://www.postgresql.org/message-id/CAJTYsWV%3DKjO5h_Pn5YGqgeoe%2B77wawQ1u00WmLOjHqcF7QdODQ%40mail.gmail.com)

**Question:** What does the postmaster log for a marked PANIC, and how does an operator tell it apart from the other children's exit-code-2 exits after a crash?

Counter-check: held. The change from "terminated by signal 6" to "exited with exit code 2" is real, and the thread does not discuss it. One sub-claim is weak: the postmaster logs only the first failing child at LOG, and the PANIC message itself still appears.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The patch marks ENOSPC errors in WAL write, sync and recovery read paths in xlog.c and xlogrecovery.c, and in walreceiver. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | ErrorData gets a no_core_dump field, and elog.h gains errabort() and errnocoredump_on_errno(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | The patch adds the developer option force_core_dump_on_panic. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Each PANIC-capable write path must be marked by hand, and nothing fails when a new path is left unmarked. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Why is matching ENOSPC at each call site better than one errfinish() rule that skips the core for a PANIC with SQLSTATE 53100?
2. Should EDQUOT and EROFS also skip the core dump, and if not, why is ENOSPC different?
3. What does the postmaster log for a marked PANIC, and how does an operator tell it apart from other children that exit with code 2?
4. Does a marked PANIC still produce core files from the other backends when send_abort_for_crash is on?
5. Are extensions expected to call errabort() and errnocoredump_on_errno(), and is that API meant to be stable?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. A committer who knows elog.c error handling and the postmaster crash-restart path would have to own it. Committers shaped the idea in 2023, but none has replied to v1 through v5.
- **Liability:** The owner signs for the choice of which PANIC sites skip the core dump, for the changed exit status of a marked PANIC, and for two new ereport() helpers that extensions may adopt. The marking list needs care in every release that adds a WAL or control-file write path.
- **Author follow-up:** The author posted five versions and answered the review comments on v2 through v4, which suggests the author stays for fixes.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs champion
- **Audience:** a sizeable minority
- **Maintenance burden:** low
- **Review cost:** medium
- **Why:** Operators who collect core dumps in production benefit, and message #1 reports the problem at fleet scale. The patch is small and its cost is limited to PANIC paths, but it touches elog.c and WAL write paths, and no committer has replied to any version.

## Appendix

### Refuted in the counter-check

None.

### Review angles

- prior-attempts: 0 objection(s). No precedent found, and nothing in the thread shows an earlier rejection. Messages #1 to #6 are the 2023 discussion that started this idea; v5 follows the direction favored there in messages #3 and #5: keep PANIC, change call sites incrementally, and control core dumps with a separate flag.
- standards-compat: 1 objection(s). Only one objection from this angle. The C API follows the errabort() spelling suggested in message #4, and the GUC is a developer option that only restores today's behaviour. The patch adds no SQL syntax and no catalog or pg_dump surface.
- replication-ondisk: 0 objection(s). The proposal touches no on-disk format, WAL record, catalog, replication protocol or wire protocol, and needs no pg_upgrade work. A marked PANIC changes only how the failing process ends: _exit(2) replaces abort(), and both skip cleanup and lead the postmaster into the same crash-restart path. Adding the ErrorData field changes the ABI only for a major release (target 20), so backpatching is not in question.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The new GUC is set only in postgresql.conf or on the command line, errabort() is a C API with no SQL surface, and fewer core files means less server memory written to disk. Unprivileged users can already cause ENOSPC; this change removes core files and does not add one.
- maintenance-fit: 1 objection(s). From the maintenance angle the design is small. Its costs fall on PANIC paths only, with no hot-path or per-backend cost, and it follows the flag approach suggested in message #4 and message #5. Committability: Andres Freund, Jeff Davis and Tom Lane shaped the idea in 2023, but no committer has replied to v1-v5. Only a non-committer reviewer is engaged, so the owner of the aftermath is not yet known.
- already-exists: 0 objection(s). No existing setting gives selective core suppression: ulimit -c, kernel core_pattern and systemd-coredump limits apply to every abort(), and filtering out SIGABRT drops too much, as message #1 says. An emit_log_hook extension could call _exit(2) on PANIC with ENOSPC, but that is a fragile workaround outside core's control, and messages #1-#6 agreed to put this in core.
- demand: 0 objection(s). The material shows real demand. Andres Freund reports fleet-wide crash analysis made harder by thousands of identical ENOSPC cores (message #1), and Jeff Davis and Tom Lane agreed and backed incremental call-site changes (messages #2 to #5). The author reports seeing the same disk-full cores in practice (message #7). The only workaround, disabling cores with ulimit or core_pattern, also drops the cores from real bugs, because every PANIC ends in SIGABRT. The cost falls mostly on the marked call sites: one developer GUC and two ereport() helpers, with no new hot-path cost.

### Precedents retrieved

- P1: [\[BUG\] Panic due to incorrect missingContrecPtr after promotion](https://commitfest.postgresql.org/patch/3675/) (cf, Withdrawn, 2022-09-21) - judged not relevant; reason summarised from its thread
- P2: [Revert "Prevent panic during shutdown checkpoint"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=703f148e98ecb4b299fdad403fc5a1de51220714) (revert, Reverted, 2017-06-06) - judged not relevant
- P3: [Deletion of the recycled future WAL files to free-up disk space if max_wal_size is reduced or wal_recycle is set to off](https://commitfest.postgresql.org/patch/3664/) (cf, Returned with feedback, 2022-08-01) - judged not relevant; reason summarised from its thread
- P4: [remove single character literal tokens from the core grammar](https://commitfest.postgresql.org/patch/2381/) (cf, Rejected, 2020-01-18) - judged not relevant; reason summarised from its thread
- P5: [Connection string usage for Core Postgresql client applications](https://commitfest.postgresql.org/patch/2354/) (cf, Withdrawn, 2020-01-21) - judged not relevant; reason summarised from its thread
- P6: [Improve pg_dump dumping publication tables](https://commitfest.postgresql.org/patch/2728/) (cf, Withdrawn, 2021-01-15) - judged not relevant; reason summarised from its thread
- P7: [pg_dump --where option](https://commitfest.postgresql.org/patch/2600/) (cf, Returned with feedback, 2020-09-30) - judged not relevant; reason summarised from its thread
- P8: [add option to pg_dumpall to exclude tables from the dump](https://commitfest.postgresql.org/patch/671/) (cf, Returned with feedback, 2016-10-03) - judged not relevant
- P9: [Parallel Dump to /dev/null](https://commitfest.postgresql.org/patch/1576/) (cf, Returned with feedback, 2018-03-22) - judged not relevant; reason summarised from its thread
- P10: [Generalize ereport_startup_progress infrastructure](https://commitfest.postgresql.org/patch/3807/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P11: [Reword docs of GUCs restart_after_crash and remove_temp_files_after_crash](https://commitfest.postgresql.org/patch/3356/) (cf, Rejected, 2022-04-05) - judged not relevant
- P12: [Revert "pg_dump: Lock all relations, not just plain tables".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=d3adaabaf7d555ec8bb1d83c43f72e79f1bf0b7d) (revert, Reverted, 2020-11-06) - judged not relevant
- P13: [Revert "postmaster: Start syslogger earlier".](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=833451552925d0175e1e15128e411ddef9a36996) (revert, Reverted, 2019-05-19) - judged not relevant
- P14: [BUG: cannot read big bytea (~600MB)](https://commitfest.postgresql.org/patch/1521/) (cf, Rejected, 2018-03-01) - judged not relevant
- P15: [Revert error-throwing wrappers for the printf family of functions.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=0c071936e94c6859afb2ec8d2c8dddf7bcdab7ee) (revert, Reverted, 2015-05-19) - judged not relevant
- P16: [Bug in query rewriter - hasModifyingCTE not getting set](https://commitfest.postgresql.org/patch/3134/) (cf, Returned with feedback, 2021-09-08) - judged not relevant
- P17: [PARALLEL flag for cursors](https://commitfest.postgresql.org/patch/1362/) (cf, Returned with feedback, 2018-02-07) - judged not relevant
- P18: [Revert "Use pselect(2) not select(2), if available, to wait in postmaster's loop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=64925603c972aa3a9f1d4c42686dba67f1a7e9d0) (revert, Reverted, 2017-04-24) - judged not relevant
- P19: [audo-detect and use -moutline-atomics compilation flag for aarch64](https://commitfest.postgresql.org/patch/2637/) (cf, Rejected, 2020-09-29) - judged not relevant
- P20: [Add Hook Functions for Disk Quota Extension](https://commitfest.postgresql.org/patch/1883/) (cf, Returned with feedback, 2020-07-31) - judged not relevant
- P21: [Revert "isolationtester: don't repeat the is-it-waiting query when retrying a step."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=dca369320f6023b55feb49f281d394181fc57903) (revert, Reverted, 2016-02-12) - judged not relevant
- P22: [Create a new session for postmaster in pg_ctl](https://commitfest.postgresql.org/patch/1772/) (cf, Returned with feedback, 2018-09-30) - judged not relevant
- P23: [Add regexp_positions(string text, pattern text, flags text) → setof record (starts integer\[\], lengths integer\[\])](https://commitfest.postgresql.org/patch/3027/) (cf, Returned with feedback, 2021-12-03) - judged not relevant
- P24: [Use SIGIO to detect postmaster death](https://commitfest.postgresql.org/patch/3066/) (cf, Withdrawn, 2021-07-13) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 13; wall time: 105 s; cost reported by the CLI: $1.63
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 57 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:38:03.030Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
