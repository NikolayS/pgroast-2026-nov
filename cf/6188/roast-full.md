# Design pre-review: Report oldest xmin source when autovacuum cannot remove tuples

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6188](https://commitfest.postgresql.org/patch/6188/) (Needs review); authors: Shinya Kato (shinyakato)
- Thread: [Report oldest xmin source when autovacuum cannot remove tuples](https://www.postgresql.org/message-id/flat/CAOzEurSgy-gDtwFmEbj5%2BR9PL0_G3qYB6nnzJtNStyuf87VSVg%40mail.gmail.com), started 2025-10-31
- Messages read: 55 of 55
- Patch prose read: v10-0001-Report-what-held-back-the-removable-cutoff-in-VA.patch, v1-0001-Report-oldest-xmin-source-when-autovacuum-cannot-.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium)

No objection survived at level 1: the problem is stated, several independent people ask for it, and the cost falls only on instrumented vacuums. Two fixable design points remain at level 2. maintenance-fit-1 says GetXidHorizonBlocker() duplicates horizon rules and nothing fails when they drift from ComputeXidHorizons(). standards-compat-1 says the line names one holder where other diagnostic log lines list all. Both need a decision from someone who knows the ProcArray and slot horizon code.

No level of the review pyramid fails.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: passed
- Levels 3-5 (implementation): out of scope

Provisional: Without the code, this review could not check that the slot xmin and per-database filters in GetXidHorizonBlocker() match ComputeXidHorizons(). It also could not check the tie-break among snapshot holders or who can see the pid in VERBOSE output.

## The proposal, restated

**Problem.** When VACUUM leaves dead tuples behind, its log gives the removable cutoff but not what held it back. Users must check pg_stat_activity, pg_prepared_xacts and pg_replication_slots by hand, and by then the holder may be gone. This hurts DBAs who diagnose table bloat after the fact.

**Proposed design.** v10 adds GetXidHorizonBlocker(), which scans the ProcArray and replication slots once, right after vacuum computes its cutoffs, and only for VACUUM (VERBOSE) or a logged autovacuum. It compares each proc's raw xid and raw xmin against the final horizon and reports the xid owner before a snapshot holder. VACUUM prints the result after the removable cutoff line when dead tuples were left unremovable.

**User-visible surface.**
- New VACUUM (VERBOSE) and autovacuum log line: "removable cutoff was held back by: ...", for example "prepared transaction" or "transaction holding snapshot (pid = %d)".
- The line names hot standby feedback only when the walsender holds the xmin without a slot, and names slots of any type by slot name.
- New C function GetXidHorizonBlocker() in procarray.h, which extensions can call.

**What the author claims.**
- Logs keep a durable record for analysis after the fact, which a view cannot give once the holder is gone.
- A separate scan against the final horizon avoids the wrong-PID result that tracking inside ComputeXidHorizons() produced.
- One extra shared ProcArrayLock scan per instrumented vacuum costs a few microseconds with 1000 backends.
- The report is best-effort, so the line uses the past tense.

**Thread state.** Many reviewers support the idea, and Japin Li, wenhui qiu and Laurenz Albe gave LGTM on v5. After Fujii Masao's offline comments, v10 narrowed the scope and moved to a single scan right after the cutoffs. Open points: wenhui qiu wants the active versus idle-in-transaction distinction back, Bharath's in-loop capture is still proposed, and nobody has reviewed v10 in the thread.

## 2. Prior attempts

No precedent found (corpus searched for items before 2025-10-31; 24 candidates retrieved, none is the same or a similar idea).

## 3. Objections a reviewer is likely to raise

### 1. Second copy of horizon rules can drift from ComputeXidHorizons()

fixable · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

A later release adds a new horizon source, as retain_dead_tuples added pg_conflict_detection, and updates ComputeXidHorizons() and the slot code but not GetXidHorizonBlocker(). Nothing fails. VACUUM then prints no blocker or names the wrong one, and DBAs find out during a bloat incident.

GetXidHorizonBlocker() must match each proc's raw xid and xmin, slot effective_xmin and catalog_xmin, invalidation and per-database rules exactly as the horizon code does. Message #39 raised the drift risk, and message #42 shared only the per-database filter. Message #53 shows v9 already got one slot case wrong. The slot rules and any future sources stay duplicated, and no test fails when they diverge.

Thread: [#39 (Kyotaro Horiguchi, 2026-06-29)](https://www.postgresql.org/message-id/20260629.172207.542761264338459554.horikyota.ntt%40gmail.com), [#42 (Shinya Kato, 2026-07-07)](https://www.postgresql.org/message-id/CAOzEurQSYUD64pGnbwgxHb_xUhHsO486Y1ZdV%3DdJugu922q%2BhA%40mail.gmail.com), [#53 (Shinya Kato, 2026-10-01)](https://www.postgresql.org/message-id/CAOzEurQwpdKzfvoHNoGSon%3DB7gtXfirAZvK9G6x%3DU6Qp_nNANg%40mail.gmail.com)

**Question:** Which horizon inputs does v10 share with ComputeXidHorizons() and the slot code, and what fails when a future horizon source is not added to GetXidHorizonBlocker()?

Counter-check: held. The thread shows a shared helper only for the per-database filter in v8, and the slot rules stay duplicated. The cost is a missing or wrong log line in a best-effort report, so it is fixable, but nothing fails loudly on drift.

### 2. One reported blocker hides other holders at the same cutoff

fixable · level 2, rubric item 12 · design · `standards-compat-1`

A prepared transaction and a physical slot both hold the removable cutoff. VACUUM names only the prepared transaction, and the DBA commits it. The next vacuum, perhaps hours later on a large table, still leaves dead tuples behind and only then names the slot.

Message #52 points out that log_lock_waits, log_recovery_conflict_waits and deadlock reports list every holder, while v10 reduces the set to one entry. In message #17 the author answers that the next VACUUM will show the next blocker, and that a full list would be 20+ lines. That answer may not be enough: message #52 shows a compact summary with an extra line per co-holding class and a count of snapshot holders, so the length concern does not force a single entry. The single-line format is also what log parsers will start to depend on, so changing it in a later release costs more.

Thread: [#17 (Shinya Kato, 2026-05-22)](https://www.postgresql.org/message-id/CAOzEurSa37w_wgP7SeVicsoo5ERXc4V7XQH2xee5YM-W9uRRBA%40mail.gmail.com), [#52 (Scott Ray, 2026-08-08)](https://www.postgresql.org/message-id/o8gz3Yoq3LmnoO9oeF_3pPSlndgQmosyYKAFkiVZ60HyiA_g8cDQVs8AMkSJBRrz0-nclaPVJ2c0ilGyi3TIuAqNqjjOYzLIaJa5V65tZDg%3D%40scottray.io), [#53 (Shinya Kato, 2026-10-01)](https://www.postgresql.org/message-id/CAOzEurQwpdKzfvoHNoGSon%3DB7gtXfirAZvK9G6x%3DU6Qp_nNANg%40mail.gmail.com)

**Question:** Why should the VACUUM log name one holder when other diagnostic log lines list every holder, and would a one-line count of co-holders be enough?

Counter-check: held. A disconnected standby's slot can stay at the same xmin as a prepared transaction's xid, so the scenario is real. Message #53 defers Scott's summary to a follow-up without answering the consistency point, but later lines would be additive, so the parser argument is weak.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | GetXidHorizonBlocker() reads replication slot xmins and the walsender's xmin from hot standby feedback. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | The patch adds the exported function GetXidHorizonBlocker() to procarray.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | The VACUUM (VERBOSE) line can show a table owner the pid of another role's snapshot holder. |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | A second scan must follow the horizon rules of ComputeXidHorizons() and the slot code in every later release. |
| performance on the hot path | **yes** | Each instrumented vacuum takes one extra shared ProcArrayLock scan of the ProcArray and the slots. |
| doesn't fit what Postgres is about (extension / external tool / already solved) | no |  |

## 5. Questions the author must answer before anyone reviews code

1. Which horizon inputs does v10 share with ComputeXidHorizons() and the slot code, and what fails when a future horizon source is not added to GetXidHorizonBlocker()?
2. When a prepared transaction and a slot hold the same cutoff, why name only one, and would one extra line with a count of co-holders be enough?
3. What is the full list of category strings and detail fields that v10 prints, including whether the prepared transaction GID and standby application_name remain?
4. How does v10 choose among several snapshot holders at the same xmin, and is the choice stable across runs?
5. Does a caller without pg_read_all_stats see another role's pid in the VACUUM (VERBOSE) line?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Fujii Masao is a listed reviewer and his offline comments shaped v10. The owner needs to know the ProcArray horizon computation and the replication slot xmin rules.
- **Liability:** The owner signs for a second horizon scan that must match ComputeXidHorizons() in every later release, and for a log line format that log parsers will read.
- **Author follow-up:** The author has posted ten versions over a year, rebased through six CommitFests, and narrowed the scope after review.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** proceed
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** DBAs who diagnose bloat after the fact gain a durable record, and several reviewers report support cases. The patch is about 31 KiB in procarray.c and vacuumlazy.c, and the duplicated horizon rules need care in later releases; nobody has reviewed v10 in the thread yet.

## Appendix

### Refuted in the counter-check

- `maintenance-fit-2` Exported blocker API is shaped for one log line and will change. Refuted (no failure scenario): vacuumlazy.c calls a function defined in procarray.c, so it cannot be static, and Postgres changes such internal signatures between major versions as normal practice. The standby extension caller is hypothetical, and Scott's view is a separate uncommitted patch.

### Review angles

- prior-attempts: 0 objection(s). The precedent list has no matching proposal, so there is no earlier rejection whose reason could apply. Inside the thread, v1 computed the blocker inside ComputeXidHorizons() and reported the wrong PID (message #12); v10 avoids that with a separate scan against the final horizon (messages #43, #51), so that reason no longer applies.
- standards-compat: 1 objection(s). Other compatibility points look settled. The patch adds a VACUUM VERBOSE and autovacuum log line, with no new syntax, GUC, catalog or dump object, and existing lines keep their meaning.
- replication-ondisk: 0 objection(s). v10 adds only a VACUUM log line and an additive procarray.h function, so it creates no on-disk, WAL, catalog or wire protocol state. Slots and walsender xmin are read, never written, and recovery is excluded by assertion. Not checkable without the code: whether the slot-xmin and per-database filters still match ComputeXidHorizons() exactly.
- security: 0 objection(s). No security objection that would hold up. The new VACUUM (VERBOSE) line can tell a table owner that another role's pid holds a snapshot at the cutoff, which pg_stat_activity hides from roles without pg_read_all_stats. The xid owner's pid, slot names and prepared transactions are already visible to all users through pg_locks, pg_replication_slots and pg_prepared_xacts, and v10 dropped the session state, so the marginal leak is small. Without the code I could not check one point: whether the line is shown to callers who lack pg_read_all_stats.
- maintenance-fit: 2 objection(s). Hot-path cost was weighed: there is one shared scan per instrumented vacuum, Laurenz accepted it in message #22, and it does not affect queries. Committers Fujii Masao and Jacob Champion are engaged, so ownership is not a concern.
- already-exists: 0 objection(s). Postgres has no existing feature that records, per vacuum, what held back the removable cutoff. The current views (pg_stat_activity, pg_prepared_xacts, pg_replication_slots, pg_stat_replication) show only the present state, and the existing "cutoff is far in the past" warning names no holder. No contrib module or extension I can name does this. The view-versus-log question was raised in message #26, and messages #29, #33 and #35 settled it in favour of having both.
- demand: 0 objection(s). The material shows real demand from several independent people: Fujii Masao, Sami Imseih, Dilip Kumar, Laurenz Albe, Jacob Champion and Bharath Rupireddy. Jacob and Bharath each wrote a patch for this on their own (messages #33, #40), and Scott Ray built a related view (message #45). Bharath (#47) and wenhui qiu (#54) describe concrete support cases. The cost is limited to instrumented vacuums, so demand is not a reason to reject; a remaining question is whether v10's narrower output, without active versus idle state or GID, is still useful enough.

### Precedents retrieved

- P1: [Revert changes to CONCURRENTLY that "sped up" Xmin advance](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e28bb885196916b0a3d898ae4f2be0e38108d81b) (revert, Reverted, 2022-05-31) - judged not relevant
- P2: [Revert "Test that vacuum removes tuples older than OldestXmin"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=efcbb76efe406d59c2ba8b4a09e04c01158ba575) (revert, Reverted, 2024-07-22) - judged not relevant
- P3: [Remove size limitations of vacuums dead_tuples array](https://commitfest.postgresql.org/patch/2302/) (cf, Returned with feedback, 2019-11-30) - judged not relevant; reason summarised from its thread
- P4: [Switch to FullTransactionId for PGPROC->xid and XLogRecord->xl_xid](https://commitfest.postgresql.org/patch/4741/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P5: [Allow single table VACUUM in transaction block](https://commitfest.postgresql.org/patch/3969/) (cf, Returned with feedback, 2022-11-24) - judged not relevant; reason summarised from its thread
- P6: [Revert "Make vacuum a bit more verbose to debug BF failure."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c62a0a49f33a0d45a97aa1d3a5bc6ddc83f10d82) (revert, Reverted, 2020-08-20) - judged not relevant
- P7: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2462/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P8: [Revert "Add single-item cache when looking at topmost XID of a subtrans XID"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b4529005fd387e863bfa9eb863629b1183c0449c) (revert, Reverted, 2022-05-28) - judged not relevant
- P9: [vacuum verbose is unclear](https://commitfest.postgresql.org/patch/2425/) (cf, Returned with feedback, 2020-07-05) - judged not relevant; reason summarised from its thread
- P10: [64-bit XIDs](https://commitfest.postgresql.org/patch/3594/) (cf, Returned with feedback, 2023-12-15) - judged not relevant; reason summarised from its thread
- P11: [Replica sends an incorrect epoch in its hot standby feedback to the Master](https://commitfest.postgresql.org/patch/2463/) (cf, Withdrawn, 2020-03-03) - judged not relevant; reason summarised from its thread
- P12: [idle-in-transaction timeout error does not give a hint](https://commitfest.postgresql.org/patch/1892/) (cf, Withdrawn, 2019-03-31) - judged not relevant; reason summarised from its thread
- P13: [Switching XLog source from archive to streaming when primary available](https://commitfest.postgresql.org/patch/3663/) (cf, Returned with feedback, 2024-12-11) - judged not relevant
- P14: [64-bit XIDs](https://commitfest.postgresql.org/patch/4703/) (cf, Returned with feedback, 2024-09-16) - judged not relevant
- P15: [Add LWLock blocker(s) information](https://commitfest.postgresql.org/patch/2576/) (cf, Returned with feedback, 2020-11-18) - judged not relevant
- P16: [Track Oldest Initialized WAL Buffer Page](https://commitfest.postgresql.org/patch/4210/) (cf, Returned with feedback, 2023-08-03) - judged not relevant
- P17: [Revert "Avoid spurious deadlocks when upgrading a tuple lock"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=9d20b0ec8f2af43041b1a65e5fcd91acc47e9ace) (revert, Reverted, 2019-06-17) - judged not relevant
- P18: [(DUPLICATE) Removing dead code in pgcrypto](https://commitfest.postgresql.org/patch/3860/) (cf, Withdrawn, 2022-08-30) - judged not relevant
- P19: [Fix cancelled queries when hot_standby_feedback = on](https://commitfest.postgresql.org/patch/1683/) (cf, Returned with feedback, 2019-01-31) - judged not relevant
- P20: [Synchronize logical replication slots from primary to standby](https://commitfest.postgresql.org/patch/3393/) (cf, Returned with feedback, 2022-02-18) - judged not relevant
- P21: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - judged not relevant
- P22: [Enforce creation of destination folders for source files in pg_regress](https://commitfest.postgresql.org/patch/100/) (cf, Rejected, 2015-03-01) - judged not relevant
- P23: [Explicit relation name in VACUUM VERBOSE log](https://commitfest.postgresql.org/patch/1246/) (cf, Returned with feedback, 2017-10-02) - judged not relevant
- P24: [vacuum verbose: show pages marked allvisible/frozen/hintbits](https://commitfest.postgresql.org/patch/2430/) (cf, Returned with feedback, 2020-09-07) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 129 s; cost reported by the CLI: $3.89
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 3 from cache, 5 summarised now, 0 failed; quotes: 5 verified, 0 dropped; relevance pass: done, 0 of 24 kept
- Tone check: 60 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:44:42.301Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
