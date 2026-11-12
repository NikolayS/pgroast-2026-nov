# Design pre-review: Allow wal_log_hints to be changed without restart

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6888](https://commitfest.postgresql.org/patch/6888/) (Needs review); authors: Bertrand Drouvot (bertranddrouvot.pg)
- Thread: [Allow wal_log_hints to be changed without restart](https://www.postgresql.org/message-id/flat/ajAAwSFy0WVMroyk%40bdtpg), started 2026-06-15
- Messages read: 2 of 2
- Patch prose read: v1-0001-Allow-wal_log_hints-to-be-changed-without-restart.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The goal is understood and has reviewer support, but the design fails at level 2. Message #2 reproduces wrong tuple visibility after pg_rewind when wal_log_hints was off after the divergence and then reloaded to on (replication-ondisk-1), and the author has not replied. The proposal also does not define the value on standbys and at promotion (replication-ondisk-2). Demand is thin (demand-1, already-exists-1), but that alone would not stop review of a small patch.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, crash safety during the change, the handling of the shared flag in recovery, and the cost of the shared read in XLogHintBitIsNeeded() could not be judged.

## The proposal, restated

**Problem.** wal_log_hints needs a restart, but data checksums can now be enabled and disabled online since f19c0ecca. Users who run without data checksums and want to measure the extra WAL that checksums would cause must still restart, so the test step is the only part of the flow that needs one.

**Proposed design.** The patch changes wal_log_hints from PGC_POSTMASTER to PGC_SIGHUP. A new shared flag, XLogCtl->walLogHints, is the value all backends read through XLogHintBitIsNeeded(), and only the checkpointer writes it, with the ordering used by UpdateFullPageWrites(). On enable it sets the flag before it WAL-logs the parameter change; on disable it WAL-logs the change first, inside a critical section.

**User-visible surface.**
- GUC wal_log_hints becomes PGC_SIGHUP and can be changed in postgresql.conf or on the command line, then applied with a reload.
- Docs for wal_log_hints in config.sgml change from "only at server start" to the standard sighup wording.
- New pg_rewind TAP test 012_wal_log_hints.pl covers a reload of wal_log_hints before pg_rewind.

**What the author claims.**
- Online wal_log_hints completes an online flow with online data checksums: test the WAL cost, then enable or disable checksums.
- Procsignal barriers are not needed because the change is not a multi-state transition and backends need not agree at the same moment.
- Both failure directions of the flag and WAL record ordering are harmless, since they only cause extra WAL.
- pg_rewind's existing check on ControlFile->wal_log_hints stays sufficient, because only WAL after the fork point matters.

**Thread state.** Tatsuya Kawata supports the direction but reports a reproduced pg_rewind corruption: off after divergence, reload to on, then pg_rewind passes and the rewound node shows a row that does not exist on the source. He proposes that pg_rewind fail when its scanned WAL contains an XLOG_PARAMETER_CHANGE with wal_log_hints = off; the author has not replied.

## 2. Prior attempts

No precedent found (corpus searched for items before 2026-06-15; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. pg_rewind accepts a target that had wal_log_hints off after divergence

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

An operator turns wal_log_hints off with a reload, a failover happens, and the operator then reloads it back on. pg_rewind passes its pg_control check but misses hint-bit-only page changes made during the off period. The rewound node shows a row that does not exist on the source, and nobody finds out until queries return wrong results.

Message #2 reproduced this and noted that the patch's own 012_wal_log_hints.pl runs this exact sequence and reports success. The claim in message #1 that the current pg_control value is enough contradicts the old thread it cites, which asks for the value to be on continuously from the fork point. A restart can already create the gap, but a reload makes it a routine action. The author has not replied, and the patch adds no check on the pg_rewind side.

Thread: [#1 (Bertrand Drouvot, 2026-06-15)](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2 (Tatsuya Kawata, 2026-07-20)](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com)

**Question:** How will pg_rewind refuse a target whose WAL after the fork point contains an XLOG_PARAMETER_CHANGE with wal_log_hints off, as message #2 proposes?

Counter-check: held. Message #2 reproduced wrong visibility with the sequence that the patch's own 012_wal_log_hints.pl runs, and the author has not replied. A restart can create the same gap today, but a reload makes it much easier to hit, and a pg_rewind check on scanned WAL can fix it.

### 2. Effective value on standbys and after promotion is not defined

fixable · level 2, rubric item 14 · WAL / replication / standby · `replication-ondisk-2`

A standby has wal_log_hints = off in its own postgresql.conf and replays a primary that runs with it on. After promotion, the new primary's shared flag and pg_control may hold the local or the replayed value. A later pg_rewind of this node, or of its peer, then trusts a value that never matched the WAL it wrote.

The proposal describes only the checkpointer on a running primary. It does not say what replay of XLOG_PARAMETER_CHANGE does to XLogCtl->walLogHints or to pg_control on a standby. It also does not say when the local setting takes effect at promotion, or whether a record is written then. Today the setting is fixed at start, so these cases did not need an answer before this patch.

Thread: [#1 (Bertrand Drouvot, 2026-06-15)](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg)

**Question:** What value does a standby hold during replay and at promotion, and which record shows a switch to the local setting in WAL that pg_rewind scans?

Counter-check: held. A standby can already differ from its primary today, and end of recovery logs a parameter change, so part of this exists. But the proposal does not say how XLogCtl->walLogHints is set during recovery, on a standby reload, or at promotion.

### 3. Demand rests on one measurement use case, with no field reports

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), new GUC · `demand-1`

The patch goes in for the few clusters still running without checksums whose owners want a WAL estimate. Since 04bec894a04 that group shrinks each release. Those owners already estimate today with one planned restart or on a staging clone, so the new code path serves a one-time task.

Message #1 gives a single motivation: avoid one restart before an online checksum change. It cites no -general or -bugs report and no user who asked for this. Message #2 adds a +1 but no use case of its own. With checksums on by default, the users left are those who chose otherwise or came through an upgrade, and the material does not size that group.

Thread: [#1 (Bertrand Drouvot, 2026-06-15)](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2 (Tatsuya Kawata, 2026-07-20)](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com)

**Question:** Which users today keep checksums off, want to measure hint-bit WAL first, and cannot afford one restart or a test on a clone?

Counter-check: held. Message #1 gives one use case and no user report, and message #2 adds a +1 without a use case. The patch is small and follows the full_page_writes pattern, which lowers the cost but does not show demand.

### 4. Measuring checksum WAL cost already works without a production restart

fixable · level 1b, rubric item 2 · desirability, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `already-exists-1`

An operator wants the WAL cost of checksums on a cluster that runs without them. Today they can enable data checksums online and watch steady-state WAL, then disable them online if the cost is too high, or restart a clone with wal_log_hints = on. The patch adds a reloadable path, and with it the pg_rewind gap shown in message #2, so that this one test step avoids a restart.

Message #1 gives the measurement as the main use, and the online checksum feature from f19c0ecca can already do that without a restart. Enabling checksums online likely costs a full pass that writes every page, so the existing way may be expensive on large clusters. That makes it less convenient, but it still works. A committer would likely ask whether that gap is worth a permanent shared flag plus a new ordering rule in the checkpointer.

Thread: [#1 (Bertrand Drouvot, 2026-06-15)](https://www.postgresql.org/message-id/ajAAwSFy0WVMroyk%40bdtpg), [#2 (Tatsuya Kawata, 2026-07-20)](https://www.postgresql.org/message-id/CAHza6qegmyONRMZspxYro7hbRtp_UrFShY%3DhD6x%3DQEwAydumrw%40mail.gmail.com)

**Question:** Which users cannot measure checksum WAL cost by enabling checksums online or by testing a restarted copy, and why is that cost too high for them?

Counter-check: held. Enabling checksums online rewrites every page, so it is a poor substitute, but a restarted clone or one planned restart still gives the measurement. The thread does not show why that cost is too high for the affected users.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | The wal_log_hints value in pg_control can now change while the server runs, and pg_rewind reads it. |
| WAL / replication / standby | **yes** | The checkpointer writes XLOG_PARAMETER_CHANGE on reload, and pg_rewind correctness depends on the setting's history. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch touches xlog.h and changes what XLogHintBitIsNeeded() reads. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | **yes** | No new GUC, but wal_log_hints changes context from PGC_POSTMASTER to PGC_SIGHUP. |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | Message #2 says a restart can already create the same pg_rewind gap, so a pg_rewind check may concern released branches. |
| maintenance burden ("who maintains this in 10 years") | **yes** | A shared flag, an ordering rule in the checkpointer, and a likely new pg_rewind check on scanned WAL. |
| performance on the hot path | **yes** | XLogHintBitIsNeeded() now reads a shared-memory flag on the hint-bit path. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | One planned restart or a restarted clone already gives the WAL measurement. |

## 5. Questions the author must answer before anyone reviews code

1. How will pg_rewind refuse a target whose WAL after the divergence point contains an XLOG_PARAMETER_CHANGE record with wal_log_hints = off, as message #2 proposes?
2. What value does XLogCtl->walLogHints hold on a standby during replay, after a standby reload, and at promotion, and which WAL record shows a switch?
3. What does pg_control report for wal_log_hints between the reload and the checkpointer's update, and can pg_rewind read a stale value after a crash in that window?
4. Which users keep data checksums off, want to measure hint-bit WAL first, and cannot use one planned restart or a restarted clone?
5. How does a reload of wal_log_hints behave while data checksums are being enabled or disabled online at the same time?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. Needs a committer who knows xlog.c parameter-change logging, the checkpointer, and pg_rewind's WAL scan; none has spoken in the visible thread.
- **Liability:** The owner signs for pg_rewind correctness on any cluster that reloads wal_log_hints, and for the flag and WAL record ordering in the checkpointer, with no end date.
- **Author follow-up:** The author moved the entry to CF PG20-3 on 2026-10-04 but has not replied to message #2 from 2026-07-20.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** The patch is small and follows the full_page_writes pattern, but it serves a one-time measurement for clusters without checksums. The reproduced pg_rewind inconsistency in message #2 needs an agreed fix before code review is useful.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Earlier discussion's pg_rewind caveat still applies and is unanswered. Refuted (not specific): Duplicate of replication-ondisk-1. Both describe the same off period after divergence and the same pg_rewind check, and replication-ondisk-1 states the required fix more directly.
- `standards-compat-1` Reload breaks the meaning of wal_log_hints that pg_rewind users rely on. Refuted (not specific): Duplicate of replication-ondisk-1. A failover tool that checks the setting still runs pg_rewind afterwards, so a check inside pg_rewind covers this case too.
- `maintenance-fit-1` The pg_control wal_log_hints field no longer covers the WAL range tools read. Refuted (not specific): Duplicate of replication-ondisk-1. The material names no third-party tool that rewinds on the pg_control value alone, so the extra scenario adds nothing concrete.
- `maintenance-fit-2` A narrow measurement use case pays for a new rewind invariant. Refuted (not specific): Duplicate of demand-1 for the missing demand and of already-exists-1 for the online checksums alternative. Its cost argument repeats the pg_rewind gap in replication-ondisk-1.
- `demand-2` Measurement convenience for a few, pg_rewind risk for many. Refuted (not specific): Duplicate of replication-ondisk-1. The cost it names is the same pg_rewind gap, and a pg_rewind check on scanned WAL removes the exposure for users who never change the setting.

### Review angles

- prior-attempts: 1 objection(s). No precedent found in the list. The only earlier discussion is the thread cited in messages #1 and #2, and the objection above is based on what message #2 reports about it.
- standards-compat: 1 objection(s). The GUC context change follows the full_page_writes pattern and needs no new syntax, psql or pg_dump change. The only compatibility issue I see is the pg_rewind contract.
- replication-ondisk: 2 objection(s). Two objections are raised. The ordering of the shared flag and the WAL record on the primary looks sound. Crash safety during the change cannot be checked without the code.
- security: 0 objection(s). No security objection. As PGC_SIGHUP, wal_log_hints can still be changed only through postgresql.conf, the command line or ALTER SYSTEM, so only superusers or roles granted ALTER SYSTEM on it can change it, as before. The proposal exposes no new data and gives unprivileged users no new path to load the server. The pg_rewind correctness problem in message #2 belongs to the safety angle.
- maintenance-fit: 2 objection(s). No hot-path objection is raised: a read of a shared boolean in XLogHintBitIsNeeded() is unlikely to cost much, and that cannot be checked without the code.
- already-exists: 1 objection(s). One objection from this angle. No extension or contrib module makes a postmaster GUC reloadable, so the only existing alternatives are online checksums and a restarted test copy.
- demand: 2 objection(s). Two objections raised at level 1b; the pg_rewind correctness issue itself belongs to the safety angle.

### Precedents retrieved

- P1: [WAL-logging facility for pgstats kinds](https://commitfest.postgresql.org/patch/5466/) (cf, Withdrawn, 2025-02-12) - judged not relevant; reason summarised from its thread
- P2: [If pg_hba.conf has changed since last reload, emit a HINT to the client and server log](https://commitfest.postgresql.org/patch/50/) (cf, Returned with feedback, 2015-04-30) - judged not relevant; reason summarised from its thread
- P3: [New wal_level to disable WAL logging](https://commitfest.postgresql.org/patch/2807/) (cf, Rejected, 2021-07-05) - judged not relevant; reason summarised from its thread
- P4: [Add WAL recovery info such as current WAL file and WAL source to server logs](https://commitfest.postgresql.org/patch/3443/) (cf, Withdrawn, 2022-10-26) - judged not relevant; reason summarised from its thread
- P5: [Logical insert/update/delete WAL records for custom table AMs](https://commitfest.postgresql.org/patch/3394/) (cf, Withdrawn, 2022-07-20) - judged not relevant; reason summarised from its thread
- P6: [pg_rewind does not rewind diverging timelines](https://commitfest.postgresql.org/patch/6811/) (cf, Withdrawn, 2026-05-29) - judged not relevant; reason summarised from its thread
- P7: [Exit walsender before confirming remote flush in logical replication](https://commitfest.postgresql.org/patch/4085/) (cf, Returned with feedback, 2023-03-17) - judged not relevant; reason summarised from its thread
- P8: [Make pg_rewind not copy useless WALs](https://commitfest.postgresql.org/patch/1302/) (cf, Returned with feedback, 2018-01-31) - judged not relevant; reason summarised from its thread
- P9: [Reopen logfile on SIGHUP](https://commitfest.postgresql.org/patch/1558/) (cf, Returned with feedback, 2018-04-10) - judged not relevant
- P10: [Reset vacuum_defer_cleanup_age to PGC_SIGHUP. Revert commit 84725aa5efe11688633b553e58113efce4181f2e](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=f480e294498533820f3ef3e6de4dcb8ff5401140) (revert, Partially reverted, 2013-02-04) - judged not relevant
- P11: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant
- P12: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P13: [Data-only mode for pg_rewind](https://commitfest.postgresql.org/patch/2063/) (cf, Returned with feedback, 2019-08-01) - judged not relevant
- P14: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P15: [Add tablespace tap test to pg_rewind](https://commitfest.postgresql.org/patch/2059/) (cf, Returned with feedback, 2019-05-10) - judged not relevant
- P16: [Reload SSL certificates on SIGHUP](https://commitfest.postgresql.org/patch/259/) (cf, Returned with feedback, 2015-10-03) - judged not relevant
- P17: [Full support for index LP_DEAD hint bits on standby](https://commitfest.postgresql.org/patch/2947/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P18: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - judged not relevant
- P19: [Shared-memory based stats collector](https://commitfest.postgresql.org/patch/1708/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P20: [Shared Memory Context](https://commitfest.postgresql.org/patch/2166/) (cf, Returned with feedback, 2019-09-30) - judged not relevant
- P21: [NUMA shared memory interleaving](https://commitfest.postgresql.org/patch/5703/) (cf, Returned with feedback, 2025-07-07) - judged not relevant
- P22: [Start WAL streaming as early as possible, without waiting for startup process to finish replaying existing WAL. This helps synchronous replication to resume quickly when significant replay lag has been built up.](https://commitfest.postgresql.org/patch/2676/) (cf, Returned with feedback, 2020-12-01) - judged not relevant
- P23: [invalid bit/varbit input causes the log file to be unreadable](https://commitfest.postgresql.org/patch/2618/) (cf, Withdrawn, 2020-06-30) - judged not relevant
- P24: [Shared Memory Context](https://commitfest.postgresql.org/patch/2325/) (cf, Returned with feedback, 2020-02-01) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 163 s; cost reported by the CLI: $1.65
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:06:47.697Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
