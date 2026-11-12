# Design pre-review: Report oldest xmin source when autovacuum cannot remove tuples

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

[CommitFest 6188](https://commitfest.postgresql.org/patch/6188/); [thread](https://www.postgresql.org/message-id/flat/CAOzEurSgy-gDtwFmEbj5%2BR9PL0_G3qYB6nnzJtNStyuf87VSVg%40mail.gmail.com), 55 of 55 messages read; no code read.

## Verdict

**NEEDS-COMMITTER-EYES** (confidence: medium). No objection survived at level 1: the problem is stated, several independent people ask for it, and the cost falls only on instrumented vacuums. Two fixable design points remain at level 2. maintenance-fit-1 says GetXidHorizonBlocker() duplicates horizon rules and nothing fails when they drift from ComputeXidHorizons(). standards-compat-1 says the line names one holder where other diagnostic log lines list all. Both need a decision from someone who knows the ProcArray and slot horizon code.

Not checked: Without the code, this review could not check that the slot xmin and per-database filters in GetXidHorizonBlocker() match ComputeXidHorizons(). It also could not check the tie-break among snapshot holders or who can see the pid in VERBOSE output.

## Prior attempts

None found before 2025-10-31.

## Objections a reviewer is likely to raise

### 1. Second copy of horizon rules can drift from ComputeXidHorizons()

A later release adds a new horizon source, as retain_dead_tuples added pg_conflict_detection, and updates ComputeXidHorizons() and the slot code but not GetXidHorizonBlocker(). Nothing fails. VACUUM then prints no blocker or names the wrong one, and DBAs find out during a bloat incident.

GetXidHorizonBlocker() must match each proc's raw xid and xmin, slot effective_xmin and catalog_xmin, invalidation and per-database rules exactly as the horizon code does. Message #39 raised the drift risk, and message #42 shared only the per-database filter. Message #53 shows v9 already got one slot case wrong. The slot rules and any future sources stay duplicated, and no test fails when they diverge.

**Question:** Which horizon inputs does v10 share with ComputeXidHorizons() and the slot code, and what fails when a future horizon source is not added to GetXidHorizonBlocker()?

<sub>fixable, level 2, maintenance burden ("who maintains this in 10 years"); messages [#39](https://www.postgresql.org/message-id/20260629.172207.542761264338459554.horikyota.ntt%40gmail.com), [#42](https://www.postgresql.org/message-id/CAOzEurQSYUD64pGnbwgxHb_xUhHsO486Y1ZdV%3DdJugu922q%2BhA%40mail.gmail.com), [#53](https://www.postgresql.org/message-id/CAOzEurQwpdKzfvoHNoGSon%3DB7gtXfirAZvK9G6x%3DU6Qp_nNANg%40mail.gmail.com); `maintenance-fit-1`</sub>

### 2. One reported blocker hides other holders at the same cutoff

A prepared transaction and a physical slot both hold the removable cutoff. VACUUM names only the prepared transaction, and the DBA commits it. The next vacuum, perhaps hours later on a large table, still leaves dead tuples behind and only then names the slot.

Message #52 points out that log_lock_waits, log_recovery_conflict_waits and deadlock reports list every holder, while v10 reduces the set to one entry. In message #17 the author answers that the next VACUUM will show the next blocker, and that a full list would be 20+ lines. That answer may not be enough: message #52 shows a compact summary with an extra line per co-holding class and a count of snapshot holders, so the length concern does not force a single entry. The single-line format is also what log parsers will start to depend on, so changing it in a later release costs more.

**Question:** Why should the VACUUM log name one holder when other diagnostic log lines list every holder, and would a one-line count of co-holders be enough?

<sub>fixable, level 2, design; messages [#17](https://www.postgresql.org/message-id/CAOzEurSa37w_wgP7SeVicsoo5ERXc4V7XQH2xee5YM-W9uRRBA%40mail.gmail.com), [#52](https://www.postgresql.org/message-id/o8gz3Yoq3LmnoO9oeF_3pPSlndgQmosyYKAFkiVZ60HyiA_g8cDQVs8AMkSJBRrz0-nclaPVJ2c0ilGyi3TIuAqNqjjOYzLIaJa5V65tZDg%3D%40scottray.io), [#53](https://www.postgresql.org/message-id/CAOzEurQwpdKzfvoHNoGSon%3DB7gtXfirAZvK9G6x%3DU6Qp_nNANg%40mail.gmail.com); `standards-compat-1`</sub>

## Questions for the author

1. Which horizon inputs does v10 share with ComputeXidHorizons() and the slot code, and what fails when a future horizon source is not added to GetXidHorizonBlocker()?
2. When a prepared transaction and a slot hold the same cutoff, why name only one, and would one extra line with a count of co-holders be enough?
3. What is the full list of category strings and detail fields that v10 prints, including whether the prepared transaction GID and standby application_name remain?
4. How does v10 choose among several snapshot holders at the same xmin, and is the choice stable across runs?
5. Does a caller without pg_read_all_stats see another role's pid in the VACUUM (VERBOSE) line?

## Touch points

- **WAL / replication / standby:** GetXidHorizonBlocker() reads replication slot xmins and the walsender's xmin from hot standby feedback.
- **ABI / extension API:** The patch adds the exported function GetXidHorizonBlocker() to procarray.h.
- **security / privileges:** The VACUUM (VERBOSE) line can show a table owner the pid of another role's snapshot holder.
- **maintenance burden ("who maintains this in 10 years"):** A second scan must follow the horizon rules of ComputeXidHorizons() and the slot code in every later release.
- **performance on the hot path:** Each instrumented vacuum takes one extra shared ProcArrayLock scan of the ProcArray and the slots.

## Who would commit it, and when

- Owner: qualified committer engaged. Fujii Masao is a listed reviewer and his offline comments shaped v10. The owner needs to know the ProcArray horizon computation and the replication slot xmin rules.
- Priority: proceed; audience a sizeable minority; review cost medium. DBAs who diagnose bloat after the fact gain a durable record, and several reviewers report support cases. The patch is about 31 KiB in procarray.c and vacuumlazy.c, and the duplicated horizon rules need care in later releases; nobody has reviewed v10 in the thread yet.

---

[roast-full.md](roast-full.md) has 0 more objection(s) that held, 1 refuted, the proposal restated, all 15 touch points, run details.

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
