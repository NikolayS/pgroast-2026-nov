# Design pre-review: Expose checkpoint timestamp and duration in pg_stat_checkpointer

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6307](https://commitfest.postgresql.org/patch/6307/) (Needs review); authors: Soumya S Murali (soumyamurali)
- Thread: [\[PATCH\] Expose checkpoint timestamp and duration in pg_stat_checkpointer](https://www.postgresql.org/message-id/flat/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), started 2025-11-24
- Messages read: 15 of 15
- Patch prose read: 0001-Improve-checkpoint-logging-and-expose-last-checkpoin.patch, 0001-Added-enhancement-related-to-checkpoint-reason-durat.patch, 0001-Enhance-checkpoint-logs-to-include-reason-manual-tim.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The review stops at level 1b. The thread cites no monitoring tool or operator that asked for last-only gauges (demand-1). pg_control_checkpoint(), the num_* counters and the total= field of the log line already give the same data (already-exists-1). The "checkpoint starting:" line already prints the checkpoint flags (already-exists-2). The log change has support in message #2, so the goal may be fine once the author answers these points.

First failing level, from the base up: **Level 1b: desirability**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: failed
- Level 2: architecture and design: not evaluated
- Levels 3-5 (implementation): out of scope

Provisional: No full, split patch set was available. Without the code, this review could not check how reset, flush and restartpoints treat the gauges, or which flags the complete line prints.

## The proposal, restated

**Problem.** Monitoring tools that read pg_stat_checkpointer see only cumulative counters such as write_time and sync_time, so they cannot easily spot one slow checkpoint or unusually frequent checkpoints. The checkpoint complete log line also does not say which kind of checkpoint ran. The author hit this while debugging checkpointer write behavior.

**Proposed design.** The patch adds two gauges to pg_stat_checkpointer that each checkpoint overwrites: the duration of the most recent checkpoint and its end time. Separately, the checkpoint complete log line, emitted when log_checkpoints is on, names the checkpoint type, such as immediate, timed or manual. The author agreed to split these two changes into separate patches.

**User-visible surface.**
- New column last_checkpoint_duration in pg_stat_checkpointer, in milliseconds; first named checkpoint_total_time.
- New column last_checkpoint_time in pg_stat_checkpointer, a timestamp of the last checkpoint end.
- New SQL functions behind the columns, such as pg_stat_get_checkpointer_checkpoint_total_time.
- The checkpoint complete log message gains the checkpoint type in parentheses, for example "checkpoint complete (immediate)".
- Fields added to the PgStat_CheckpointerStats struct in pgstat.h.

**What the author claims.**
- A monitoring agent can alert when last_checkpoint_duration jumps, for example from about 300 ms to 5000 ms, even if several checkpoints ran between samples.
- last_checkpoint_time lets tools detect checkpoints that run closer together than expected.
- These signals are hard to get from cumulative write_time and sync_time without complex delta calculations.
- The new columns complement the cumulative fields and replace nothing.
- Many existing monitoring dashboards lack such checkpoint signals.

**Thread state.** Michael Banck supports the log change but doubts the value of last-only gauges, because a poller misses checkpoints between samples and no other view stores a last duration. Álvaro Herrera does not want to block it but asked for a sleep-time breakdown; the author's May 2026 revive got no replies and the patch needs a rebase.

## 2. Prior attempts

### Add last_vacuum_index_scans in pg_stat_all_tables (similar idea)

- **Link:** [https://commitfest.postgresql.org/patch/3756/](https://commitfest.postgresql.org/patch/3756/)
- **Thread:** [https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com)
- **Date:** decided 2023-01-31; started 2022-07-04
- **Outcome:** Withdrawn (cf)
- **Who objected:** Alvaro Herrera, Fujii Masao (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Alvaro Herrera objected that the column keeps only the last vacuum's value, unlike other stats counters, which are all additive; he and Peter Geoghegan said a history of vacuum runs would be more useful. Fujii Masao asked whether one global counter in a pg_stat_vacuum view would be enough, and Alvaro later doubted the tuning value because autovacuum memory is capped at 1 GB. The author did not resolve these points, and the visible thread ends with a request for a rebased patch that got no reply before the entry was withdrawn.
- **Reason (quote):** "No other stats counter works that way; they are all additive." ([message](https://www.postgresql.org/message-id/20220708164052.bp62uhma56v7u7n5%40alvherre.pgsql))
- **What is different this time:** Same pattern of a last-run-only value in a cumulative stats view, applied to checkpoint duration instead of vacuum index scans; the additive-stats objection is not answered.

### add the last checkpoint kind to the control file (related)

- **Link:** [https://commitfest.postgresql.org/patch/3493/](https://commitfest.postgresql.org/patch/3493/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CALj2ACUSED8QQWDHZ6yPusH7p7VecQ0pNm5Oin7WwdxkyhhUWg@mail.gmail.com](https://www.postgresql.org/message-id/flat/CALj2ACUSED8QQWDHZ6yPusH7p7VecQ0pNm5Oin7WwdxkyhhUWg@mail.gmail.com)
- **Date:** decided 2022-02-03; started 2021-12-07
- **Outcome:** Withdrawn (cf)
- **Who objected:** Julien Rouhaud, Kyotaro Horiguchi, Michael Paquier (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Reviewers saw no realistic use for the kind of an already completed checkpoint and preferred a pg_stat_progress_checkpoint view for the checkpoint in progress. The control file must stay within 512 bytes, so its space is kept for critical data, and two bytes for non-critical data were judged a likely future regret. The stored flags also leave out flags added when a running checkpoint is upgraded, for example by a later CHECKPOINT command. The author agreed to drop the patch.
- **Reason (quote):** "For now we have some room, but we will likely keep consuming bytes in that file for more critical features and it's almost certain that at one point we will regret wasting 2 bytes for that." ([message](https://www.postgresql.org/message-id/20220131055416.iq7yknigyuxcm6tj%40jrouhaud))
- **What is different this time:** Reports the last checkpoint kind in the completion log line instead of the control file, so the control-file space concern does not apply; the upgraded-flags issue may still apply.


## 3. Objections a reviewer is likely to raise

### 1. No user demand shown for last-only checkpoint gauges

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

A monitoring vendor polls pg_stat_checkpointer every minute while checkpoints run every 20 seconds during a write spike. Its alert sees one duration out of three and misses the slow one. Everyone then keeps two new columns and functions that the target users cannot rely on.

The only use case offered is the author's own debugging session (message #1), plus a hypothetical agent in message #14. No field report, -general thread or monitoring tool asking for this is cited. Michael Banck stayed unconvinced (messages #2, #11, #13), Álvaro Herrera wanted a sleep-time breakdown first (message #3), and the May 2026 revive (message #15) got no reply. [P4] was withdrawn after a similar last-only column met the objection that stats counters are additive.

**Precedents.**
- [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (Withdrawn, 2023-01-31); thread: [link](https://www.postgresql.org/message-id/flat/ce4a6ee93d78e2c4dafe4c1e515b342b@oss.nttdata.com)

Thread: [#1 (Soumya S Murali, 2025-11-24)](https://www.postgresql.org/message-id/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), [#2 (Michael Banck, 2025-11-24)](https://www.postgresql.org/message-id/20251124091822.GC8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#3 (Álvaro Herrera, 2025-11-24)](https://www.postgresql.org/message-id/202511240955.vt3fjrb4ksrs%40alvherre.pgsql), [#11 (Michael Banck, 2025-11-28)](https://www.postgresql.org/message-id/20251128080849.GA13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#13 (Michael Banck, 2025-12-01)](https://www.postgresql.org/message-id/20251201081548.GE13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#14 (Soumya S Murali, 2025-12-04)](https://www.postgresql.org/message-id/CAMtXxw_JK-YbpM9fgqnv9eAFEKoGx38Aq_tNFs0jCZL4C_4nyg%40mail.gmail.com), [#15 (Soumya S Murali, 2026-05-15)](https://www.postgresql.org/message-id/CAMtXxw9n%3DgOKYti2ekvPJ7tS7MQewgrkSmO8do-A%2BKwXXdoLLw%40mail.gmail.com)

**Question:** Which monitoring tool or operator has asked for a last-checkpoint duration, and why do cumulative counters plus log_checkpoints not meet that need?

Counter-check: held. Álvaro calls the value useful in message #3, but the thread cites no field report or tool request, and the May 2026 revive got no reply. [P4] concerns design and must not be used as support for a demand objection.

### 2. Per-checkpoint duration and time are already in logs and pg_control_checkpoint()

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved), catalog changes, maintenance burden ("who maintains this in 10 years") · `already-exists-1`

A monitoring tool reads last_checkpoint_time from pg_stat_checkpointer and checkpoint_time from pg_control_checkpoint(). The first is the checkpoint end and the second is its start, so they never match. Users find out when dashboards disagree, and they have two "last checkpoint" sources to keep apart.

pg_control_checkpoint() already returns the time of the latest checkpoint or restartpoint. The "checkpoint complete" line, logged by default under log_checkpoints, already gives total= per checkpoint, which pgbadger reads (message #7, #8). A poller can detect frequency from num_timed, num_requested and num_done. The author says this is "hard to get" without delta math (message #14), but that describes less convenient, not impossible.

Thread: [#7 (Juan José Santamaría Flecha, 2025-11-26)](https://www.postgresql.org/message-id/CAC%2BAXB1Vco9jSmjZsdGupqiRMSDfTBu_H6r%3DFQhmZrUpwScjhA%40mail.gmail.com), [#8 (Michael Banck, 2025-11-26)](https://www.postgresql.org/message-id/20251126174138.GE8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#13 (Michael Banck, 2025-12-01)](https://www.postgresql.org/message-id/20251201081548.GE13635%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#14 (Soumya S Murali, 2025-12-04)](https://www.postgresql.org/message-id/CAMtXxw_JK-YbpM9fgqnv9eAFEKoGx38Aq_tNFs0jCZL4C_4nyg%40mail.gmail.com)

**Question:** What can a tool do with the two new columns that it cannot do with pg_control_checkpoint(), the num_* counters and the log_checkpoints total= field?

Counter-check: held. pg_control_checkpoint() does return the last checkpoint time, and the complete line carries total= for every checkpoint. Message #14 says the existing route is harder, which does not show that it cannot do the job.

### 3. Checkpoint type is already logged in the checkpoint starting line

fixable · level 1b, rubric item 12 · doesn't fit what Postgres is about (extension / external tool / already solved) · `already-exists-2`

With log_checkpoints on, each checkpoint already emits "checkpoint starting: immediate force wait" or "checkpoint starting: wal". The patch repeats a subset of those flags on the complete line. Log parsers that match the complete line must change, and the two lines can show different flag sets.

Michael Banck and the author both treat the type as missing from the log (message #2, #5), but the starting line lists the flags. Pairing a start line with the next complete line is a documented, modest step that pgbadger already performs. The proposal names only manual, timed and immediate, which is fewer than the starting line shows, such as wal and end-of-recovery.

Thread: [#1 (Soumya S Murali, 2025-11-24)](https://www.postgresql.org/message-id/CAMtXxw_W6w2Q1QsCvMPnoq3xCMKzH28Zjk_EmL60oP%2BsCTkXOw%40mail.gmail.com), [#2 (Michael Banck, 2025-11-24)](https://www.postgresql.org/message-id/20251124091822.GC8364%40p46.dedyn.io%3Blightning.p46.dedyn.io), [#5 (Soumya S Murali, 2025-11-26)](https://www.postgresql.org/message-id/CAMtXxw8%2BW0UCCXS71_u3TiFnO_f47aRQPq_ar8EJspapgAs5SQ%40mail.gmail.com)

**Question:** What does the type on the complete line give that the existing checkpoint starting line, which already lists the flags, does not?

Counter-check: held. The "checkpoint starting:" line does print the flags, and the thread never weighs it against the new text. The claim that pgbadger already pairs the two lines is not in the material and should not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | no |  |
| WAL / replication / standby | **yes** | The gauges are set where restartpoints also end, so standbys are affected and the proposal does not say how. |
| wire protocol | no |  |
| catalog changes | **yes** | The patch adds functions to pg_proc.dat and columns to pg_stat_checkpointer in system_views.sql. |
| ABI / extension API | **yes** | The patch adds fields to the PgStat_CheckpointerStats struct in pgstat.h. |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | Overwritten fields sit in a struct of summed counters, and the changed log line reaches external log parsers. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | pg_control_checkpoint(), the num_* counters and the log_checkpoints lines already expose the last checkpoint time, duration and flags. |

## 5. Questions the author must answer before anyone reviews code

1. Which monitoring tool or operator has asked for a last-checkpoint duration, and why do cumulative counters plus log_checkpoints not meet that need?
2. What can a tool do with the two new columns that it cannot do with pg_control_checkpoint(), the num_* counters and the log_checkpoints total= field?
3. What does the type on the complete line give that the existing "checkpoint starting:" line, which already lists the flags, does not?
4. Why is a last-only gauge better than an additive form, such as a cumulative total duration that a poller can turn into deltas?
5. On a standby, after promotion and after pg_stat_reset_shared('checkpointer'), what do last_checkpoint_duration and last_checkpoint_time show?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. It needs a committer who knows the cumulative statistics code and the checkpointer. Álvaro Herrera and Michael Banck commented early, but nobody replied to the May 2026 revive.
- **Liability:** The owner signs for two columns with set semantics in a summed stats struct, their reset and restartpoint behaviour, and a log line format that external parsers read, for every later release.
- **Author follow-up:** The author agreed to split the patch, revived the thread in May 2026 and keeps moving the CommitFest entry forward, but no rebased, split patch set has been posted.

**Not now / show demand: yes.** This is an outcome, not a rejection. Evidence that would change it: A request from a monitoring tool or operators for a last-checkpoint duration would change it. So would a concrete checkpoint problem that counter deltas, pg_control_checkpoint() and the log_checkpoints lines cannot detect.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** needs design rethink
- **Audience:** niche
- **Maintenance burden:** medium
- **Review cost:** low
- **Why:** The patch is about 7 KiB and touches xlog.c and the checkpointer stats code, but the gauges have no shown demand and need their own reset and flush rules. The log change has support and could go ahead alone once it is split out and reuses the flag words of the starting line.

## Appendix

### Held up in the counter-check, not listed above

- `prior-attempts-1` (fixable, level 2) Last-only gauges repeat a design rejected for vacuum stats. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-1` (fixable, level 2) Changed checkpoint complete log line can break existing log parsers. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `standards-compat-2` (fixable, level 2) New checkpoint type words differ from the flags in the starting line. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `replication-ondisk-1` (fixable, level 2) Restartpoints on a standby are not addressed by the new gauges. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).
- `maintenance-fit-2` (fixable, level 2) A gauge in a summed stats struct needs its own flush and reset rules. Not listed: level 2 is above the failing level 1b: not evaluated (base-up stop rule).

### Refuted in the counter-check

- `prior-attempts-2` Logged checkpoint type may not match how the checkpoint ran. Refuted (not specific): Duplicate of standards-compat-2, which covers both the unnamed WAL-triggered case and the flags added when a later request upgrades a running checkpoint.
- `standards-compat-3` Last-only duration gauge departs from the cumulative stats view convention. Refuted (not specific): Duplicate of prior-attempts-1 for the last-only convention and the sampling gap. Its reset and restartpoint questions are covered by maintenance-fit-2 and replication-ondisk-1.
- `replication-ondisk-2` Overwrite-only gauges do not fit the additive cumulative stats model. Refuted (not specific): Duplicate of maintenance-fit-2, which asks the same question about reset, flush and snapshot handling of an overwritten field.
- `maintenance-fit-1` Last-only gauges set a new pattern in cumulative stats views. Refuted (not specific): Duplicate of prior-attempts-1. Both rest on message #13's question of why checkpoints alone get a last duration, on [P4], and on the same missed-sample scenario.
- `maintenance-fit-3` Changing the checkpoint complete line affects log parsers for little new data. Refuted (not specific): Duplicate of standards-compat-1 for the parser breakage and of already-exists-2 for the flags already printed on the starting line.
- `demand-2` Existing counters and log_checkpoints already cover the stated alerts. Refuted (not specific): Duplicate of already-exists-1, which makes the same case from counter deltas and log_checkpoints and adds pg_control_checkpoint().

### Review angles

- prior-attempts: 2 objection(s). The pg_stat_progress_checkpoint attempts ([P1], [P3]) and the log-message attempts ([P2], [P7]) cover a different idea, and their recorded reasons do not apply here.
- standards-compat: 3 objection(s). Objections listed above.
- replication-ondisk: 2 objection(s). The log-line change has no persistent or cross-node state. The view change adds only shared-memory stats fields and a system view, which need a catversion bump and nothing in pg_upgrade.
- security: 0 objection(s). The proposal has no real privilege or exposure aspect. pg_stat_checkpointer is already readable by all users and shows cluster-wide checkpoint counters and timings. A last duration and end timestamp add no new data about other users, sessions or query texts. The log change only adds the checkpoint type to a message that only privileged users can read.
- maintenance-fit: 3 objection(s). Objections found. Committability: no committer has taken ownership. Álvaro and Michael commented in message #3 and message #11, and the May 2026 revive in message #15 got no reply.
- already-exists: 2 objection(s). Two objections raised; both concern existing ways to get the same data.
- demand: 2 objection(s). The checkpoint type in the log line has positive feedback (messages #2, #11), so this review raises no demand objection to that change.

### Precedents retrieved

- P1: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3545/) (cf, Returned with feedback, 2023-01-31) - used in the review; reason summarised from its thread
- P2: [Add checkpoint stats of snapshot and mapping files of pg_logical dir](https://commitfest.postgresql.org/patch/3389/) (cf, Withdrawn, 2022-10-11) - used in the review; reason summarised from its thread
- P3: [Report checkpoint progress with pg_stat_progress_checkpoint](https://commitfest.postgresql.org/patch/3873/) (cf, Returned with feedback, 2022-10-12) - used in the review; reason summarised from its thread
- P4: [Add last_vacuum_index_scans in pg_stat_all_tables](https://commitfest.postgresql.org/patch/3756/) (cf, Withdrawn, 2023-01-31) - used in the review; reason summarised from its thread
- P5: [add the last checkpoint kind to the control file](https://commitfest.postgresql.org/patch/3493/) (cf, Withdrawn, 2022-02-03) - used in the review; reason summarised from its thread
- P6: [Add last failed connection error message to pg_stat_wal_receiver](https://commitfest.postgresql.org/patch/3666/) (cf, Withdrawn, 2022-10-11) - judged not relevant; reason summarised from its thread
- P7: [Emit "checkpoint skipped because system is idle" message at LOG level if log_checkpoints is set](https://commitfest.postgresql.org/patch/3505/) (cf, Withdrawn, 2022-01-07) - used in the review; reason summarised from its thread
- P8: [Force the old transactions logs cleanup even if checkpoint is skipped](https://commitfest.postgresql.org/patch/4619/) (cf, Returned with feedback, 2024-04-09) - judged not relevant; reason summarised from its thread
- P9: [Add statistics refresh materialized view](https://commitfest.postgresql.org/patch/3243/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P10: [SQL statements statistics counter view (pg_stat_sql)](https://commitfest.postgresql.org/patch/790/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P11: [Fix inconsistency in reporting checkpointer stats](https://commitfest.postgresql.org/patch/4078/) (cf, Returned with feedback, 2024-10-10) - judged not relevant
- P12: [allow to set log_statement_sample_limit - maximal duration for sampling](https://commitfest.postgresql.org/patch/2163/) (cf, Rejected, 2019-11-06) - judged not relevant
- P13: [Revert "Reduce checkpoints and WAL traffic on low activity database server"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=b8b69d89905e04b910bcd65efce1791477b45d35) (revert, Reverted, 2012-06-13) - judged not relevant
- P14: [pg_stat_activity: avoid showing state=active with wait_event=ClientRead](https://commitfest.postgresql.org/patch/3760/) (cf, Returned with feedback, 2022-11-30) - judged not relevant
- P15: [Fix checkpointer sync request queue problems](https://commitfest.postgresql.org/patch/3583/) (cf, Withdrawn, 2022-11-07) - judged not relevant
- P16: [Collecting statistics about contents of JSONB columns](https://commitfest.postgresql.org/patch/3500/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P17: [Add accumulated statistics for wait event](https://commitfest.postgresql.org/patch/2332/) (cf, Returned with feedback, 2020-08-02) - judged not relevant
- P18: [Command statistics system](https://commitfest.postgresql.org/patch/2578/) (cf, Returned with feedback, 2020-09-17) - judged not relevant
- P19: [Tab completion for ALTER COLUMN SET STATISTICS](https://commitfest.postgresql.org/patch/377/) (cf, Rejected, 2015-12-21) - judged not relevant
- P20: [Issue a log message when the backtrace logged is cut off](https://commitfest.postgresql.org/patch/3547/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P21: [Send duration output to separate files](https://commitfest.postgresql.org/patch/5125/) (cf, Returned with feedback, 2024-10-07) - judged not relevant
- P22: [Fix around conn_duration in pgbench](https://commitfest.postgresql.org/patch/3167/) (cf, Withdrawn, 2021-06-15) - judged not relevant
- P23: [Revert "pg_createsubscriber: Add log message when no publications exist to drop."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=45a7faf130176ae7c152413967c7c90068476a94) (revert, Reverted, 2025-10-15) - judged not relevant
- P24: [More stable query plans via more predictable column statistics](https://commitfest.postgresql.org/patch/434/) (cf, Returned with feedback, 2016-02-03) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 17; wall time: 197 s; cost reported by the CLI: $2.81
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 4 from cache, 4 summarised now, 0 failed; quotes: 3 verified, 0 dropped; relevance pass: done, 6 of 24 kept
- Tone check: 127 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T02:47:51.368Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
