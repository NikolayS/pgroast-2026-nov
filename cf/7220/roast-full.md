# Design pre-review: pg_rewind: Remove recovery at the start of rewind

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 7220](https://commitfest.postgresql.org/patch/7220/) (Needs review); authors: Srinath Reddy (srinath-reddy-sadipiralla)
- Thread: [pg_rewind: Remove recovery at the start of rewind](https://www.postgresql.org/message-id/flat/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com), started 2026-08-30
- Messages read: 1 of 1
- Patch prose read: v1-0001-pg_rewind-scan-an-unclean-target-s-WAL-instead-of-fo.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: low)

The goal is reasonable and comes with a measurement, but the proposal removes a safety step without saying what replaces its side effects. replication-ondisk-2 is decisive: crash recovery used to bring pg_control up to date before pg_rewind read it, and message #1 names no replacement. maintenance-fit-1 and replication-ondisk-3 raise the same kind of gap for end-of-recovery actions and for the postmaster.pid interlock. All are answerable, and only one message is visible, so confidence is low.

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: The code was not read. The patch may already derive the target timeline from WAL or check shared memory before it removes postmaster.pid; the prose does not say either.

## The proposal, restated

**Problem.** When the target was not shut down cleanly, pg_rewind starts it in single-user mode and replays all WAL from the last checkpoint before rewinding. Much of that redo goes into relation files that pg_rewind then overwrites from the source or removes. This costs time for users who rewind a crashed or immediately stopped old primary.

**Proposed design.** pg_rewind already reads target WAL from the last common checkpoint to build the page map, and the patch extends that scan to find the end of target WAL, so it no longer needs a shutdown checkpoint or minRecoveryPoint. It finds the checkpoint before the divergence point by following the record chain backwards, with a forward scan from the control-file checkpoint as fallback. The target must still be stopped, and pg_rewind checks for a running postmaster.

**User-visible surface.**
- The --no-ensure-shutdown option of pg_rewind is removed.
- pg_rewind accepts a target that was not shut down cleanly and no longer starts it in single-user mode.
- pg_rewind rejects a target with a live postmaster and removes a stale postmaster.pid whose PID no longer exists.

**What the author claims.**
- A local benchmark shows about 25% to 37% shorter run time for 100 MB to 2000 MB of uncheckpointed target data, about 1.5 times faster.
- The saving comes from skipping redo into files that pg_rewind later restores or removes, since both versions read target WAL.
- make check-world passes.

**Thread state.** Only the first message is visible, and nobody has replied. The CommitFest entry needs review, needed a rebase in September and has moved to a second CommitFest.

## 2. Prior attempts

### Fix pg_rewind false positives caused by shutdown-only WAL (related)

- **Link:** [https://commitfest.postgresql.org/patch/6041/](https://commitfest.postgresql.org/patch/6041/)
- **Thread:** [https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com)
- **Date:** decided 2026-05-01; started 2025-09-06
- **Outcome:** Withdrawn (cf)
- **Who objected:** Robert Haas (senders of the linked thread, picked by the AI summary)
- **Why it was not accepted (AI summary of the linked thread; check it there):** Robert Haas said the premise is wrong: the shutdown checkpoint updates the control file, which matters when deciding whether servers are in sync, and the server itself refuses to follow the new timeline. A clean shutdown also writes a RUNNING_XACTS record, which the patch did not skip. A later version from another poster did not address these points, and the entry was withdrawn.
- **Reason (quote):** "But I do not think we can do that, because (1) those WAL records might do important things like update the control file and (2) the server will not be OK with ignoring those WAL records even if pg_rewind decides that they are not important." ([message](https://www.postgresql.org/message-id/CA%2BTgmoYYvYShoRKK6%3D8PX-BYBarFXjrGB%2BcBBRjX-SkMfd0Nrg%40mail.gmail.com))
- **What is different this time:** P8 skipped shutdown-only records when finding the end of target WAL. This proposal reads every record but no longer runs the recovery that updated the control file, so only the control-file part of the P8 objection carries over.


## 3. Objections a reviewer is likely to raise

### 1. Target timeline and control-file fields are read without recovery updating them

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-2`

A node is promoted, writes on the new timeline, and crashes before its first checkpoint after promotion completes. Its pg_control still shows the old timeline in checkPointCopy. pg_rewind builds the target history from that stale timeline, picks the wrong divergence point or reports that no rewind is needed, and the node later follows the new primary with diverged blocks.

Crash recovery used to end with a shutdown checkpoint that brought pg_control's checkpoint, timeline and minRecoveryPoint up to date before pg_rewind read them. [P8] was withdrawn partly because WAL after divergence updates the control file, and pg_rewind's sync decision depends on that. The proposal does not say which control-file fields pg_rewind now trusts as-is and which it derives from the WAL scan.

**Precedents.**
- [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (Withdrawn, 2026-05-01); thread: [link](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com)

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com)

**Question:** For a target that crashed after promotion but before a checkpoint, which timeline does pg_rewind use, and how is it derived?

Counter-check: held. Today the checkpoint written when single-user recovery shuts down updates the timeline in pg_control before pg_rewind reads it, and the proposal names no replacement. P8 supports only the control-file point, because this patch reads those records and does not skip them.

### 2. pg_rewind gets a second end-of-WAL rule that must match the server's

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby · `maintenance-fit-1`

An old primary crashes with a torn last segment or an aborted contrecord. The pg_rewind scan stops at a different LSN than crash recovery would, so pg_rewind reports "no rewind required" or misses blocks. The server then refuses the new timeline or runs with stale pages, and the operator finds out only at startup or later.

Today the server decides where an unclean target's WAL ends, and pg_rewind reads the result from a shutdown checkpoint. The patch moves that decision into parsexlog.c, so any later change to end-of-WAL handling or to records with side effects must also be made in pg_rewind. A missed change does not fail loudly. [P8], from the same area and author, was withdrawn after a reviewer said pg_rewind's view of end of WAL cannot differ from the server's, and message #1 does not explain why that concern no longer applies.

**Precedents.**
- [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (Withdrawn, 2026-05-01); thread: [link](https://www.postgresql.org/message-id/flat/CAFC+b6rsM+WUoph-aPk5sz4cPzaQ4XkRDNwCJ5nG5+HsRQ=j8A@mail.gmail.com)

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com)

**Question:** How does the patch make sure the pg_rewind end-of-WAL point always equals the point crash recovery would reach, and what test fails if the two drift apart?

Counter-check: held. Record validation is shared xlogreader code and cannot drift, but the end-of-recovery actions live only in the server and the proposal names no test tying the two together. P8 concerned skipping records, which this patch does not do, so it is not support. An explanation or test can answer this.

### 3. Stale postmaster.pid removal by PID check alone can expose a live cluster

fixable · level 2, rubric item 7 · security / privileges, on-disk format / pg_upgrade · `replication-ondisk-3`

The data directory is on a shared volume, and a postmaster runs on it in another container or PID namespace. pg_rewind sees no such PID, removes postmaster.pid and rewrites relation files under the running server. The data is corrupted, and nobody notices until that server crashes or returns wrong rows.

Under the old design, a single-user postgres start protected the directory with the server's lock-file logic, which also checks whether the old shared memory segment is still in use. The commit message says pg_rewind removes postmaster.pid "when its PID no longer exists", which is a weaker check. The proposal does not say how it handles PID reuse or postmasters in another namespace.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com)

**Question:** What interlock beyond a PID existence check stops pg_rewind from removing postmaster.pid and writing to a data directory that a postmaster in another namespace still uses?

Counter-check: held. The server's lock-file check has the same gap for a separate PID namespace, so the container scenario is not new. The shared memory check does differ: a dead postmaster with live orphaned backends passes a PID-only check, and the proposal names no interlock.

### 4. Removing --no-ensure-shutdown breaks existing failover scripts

fixable · level 2, rubric item 12 · design, maintenance burden ("who maintains this in 10 years") · `standards-compat-1`

A failover tool such as a cluster manager runs pg_rewind --no-ensure-shutdown after promoting a standby. After the upgrade, pg_rewind exits with an unknown-option error. The old primary is not rejoined, and the operator finds out during the next failover, when the cluster runs without a standby.

The commit message removes the option outright because pg_rewind no longer starts the target. Users who pass it asked for exactly what the new code does: skip the single-user start. Postgres usually keeps such options as accepted no-ops for some releases, or deprecates them first. The touched files include no documentation, so the reference page would still list the option and the clean-shutdown requirement.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com)

**Question:** Why can --no-ensure-shutdown not stay as an accepted no-op, with the pg_rewind documentation updated to match the new behaviour?

Counter-check: held. The proposal gives no reason for removing the option instead of keeping it as a no-op, and the script failure is concrete. The claim that Postgres usually keeps such options is overstated, since major releases have removed client options before.

### 5. No field evidence that crash recovery dominates real rewind time

fixable · level 1b, rubric item 3 · demand, doesn't fit what Postgres is about (extension / external tool / already solved), maintenance burden ("who maintains this in 10 years") · `demand-1`

An operator rewinds a crashed old primary during failover. The rewind may finish a few seconds sooner, but every rewind now relies on pg_rewind's own end-of-WAL scan instead of the server's recovery code. If that scan mishandles a case that server recovery handles, the operator finds out when the rewound node fails to start or diverges.

The only evidence is one local benchmark in message #1, and nobody has replied. It saves under 1.5 s for 100 MB and about 19 s for 2000 MB. Message #1 gives no workload, storage, or redo share, and cites no user report that this step slows failover. The commit message drops the dependency on a shutdown checkpoint and minRecoveryPoint, so the new path may also cover clean-shutdown targets, which gain nothing.

Thread: [#1 (Srinath Reddy Sadipiralla, 2026-08-30)](https://www.postgresql.org/message-id/CAFC%2Bb6pztjF8AiM%3DGOs%3DSeNOYLpZnUSSMJsRKKZdf-MKJj22VQ%40mail.gmail.com)

**Question:** Which production failovers spend a large share of pg_rewind time in single-user crash recovery, and how many seconds would this save there?

Counter-check: held. Message #1 offers one local benchmark without workload or storage details and cites no user report, so demand is not shown. The remark about clean-shutdown targets is speculation and should be dropped.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | pg_rewind reads pg_control and rewrites data files of an unclean target without recovery having updated the control file first. |
| WAL / replication / standby | **yes** | pg_rewind now finds the end of target WAL and the pre-divergence checkpoint by its own scan. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | no |  |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | **yes** | see objection replication-ondisk-3: Stale postmaster.pid removal by PID check alone can expose a live cluster |
| backpatchability | no |  |
| maintenance burden ("who maintains this in 10 years") | **yes** | pg_rewind takes over work that server crash recovery did, and the patch removes the --no-ensure-shutdown option. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | see objection demand-1: No field evidence that crash recovery dominates real rewind time |

## 5. Questions the author must answer before anyone reviews code

1. For a target that crashed after promotion but before its first checkpoint, which timeline and checkpoint does pg_rewind use, and how does it derive them?
2. Which control-file fields does pg_rewind now trust as read, and which does it derive from the WAL scan?
3. Which end-of-recovery actions of the server does the scan not perform, and what test fails if the scan and crash recovery reach different end points?
4. What interlock beyond a PID existence check stops pg_rewind from removing postmaster.pid when the postmaster is dead but orphaned backends still run?
5. Why can --no-ensure-shutdown not stay as an accepted no-op, with the pg_rewind documentation updated to match?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** silent. The owner needs to know crash recovery, xlogreader and timeline handling as well as pg_rewind. Nobody has replied in the visible thread.
- **Liability:** The owner signs for the correctness of every rewind of an unclean target, with no time limit. A wrong timeline or end point shows up later as a standby that fails to start or holds diverged blocks.
- **Author follow-up:** The author moved the entry to the next CommitFest in October. The thread shows no rebased version after the CFbot notice in September.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** Rewinding a crashed old primary is a common use of pg_rewind, and the benchmark shows 25% to 37% shorter runs. The patch is about 27 KiB in pg_rewind only, but review needs recovery knowledge, and the control-file and interlock questions come before code review.

## Appendix

### Refuted in the counter-check

- `prior-attempts-1` Skipping redo leaves the target control file stale, as in P8. Refuted (not specific): Duplicate of replication-ondisk-2, which states the same promoted-then-crashed scenario and names the stale control-file field.
- `standards-compat-2` pg_rewind deleting postmaster.pid replaces the server's own lock check. Refuted (not specific): Duplicate of replication-ondisk-3, which also names the shared memory check that the single-user start performs today.
- `replication-ondisk-1` End of target WAL now comes from a frontend scan, not server recovery. Refuted (no failure scenario): pg_rewind reads WAL with the same xlogreader validation that crash recovery uses, so both stop at the same first invalid record. Under the WAL-before-data rule, data files hold no changes from records beyond valid flushed WAL, so no block is missed.
- `maintenance-fit-2` Removing --no-ensure-shutdown breaks existing failover scripts. Refuted (not specific): Duplicate of standards-compat-1.
- `maintenance-fit-3` No committer who knows recovery has looked at this yet. Refuted (not specific): A thread with no replies yet describes any new patch and belongs in the committability field. The rubric says weak committability must not count against the design.

### Review angles

- prior-attempts: 1 objection(s). P4, P6 and P9 also aimed to speed up pg_rewind, but they ended without review answers and recorded no design reason that applies here. P7 shares only the area.
- standards-compat: 2 objection(s). Other contract points, such as control-file state after rewind, are left to the safety angle.
- replication-ondisk: 3 objection(s). Not applicable; objections listed.
- security: 0 objection(s). The proposal has no privilege or exposure aspect: pg_rewind runs as the data directory owner, and the patch adds no role, function, view or setting. Removing a postmaster.pid whose PID is not visible, for example from another container or host on shared storage, is a data-safety risk and belongs to the safety angle.
- maintenance-fit: 3 objection(s). Not applicable.
- already-exists: 0 objection(s). No existing feature, option or extension gives the same result. Today the only path for an unclean target is the single-user crash recovery that pg_rewind runs itself, and the patch speeds up that path instead of adding a second one. End-of-WAL detection reuses the WAL scan pg_rewind already does with the shared xlogreader, so a duplicated mechanism is not apparent from the prose; safety and option removal belong to other angles.
- demand: 1 objection(s). Rewinding a crashed old primary is a common pg_rewind use, so the problem is real. Demand for this speedup has not been shown.

### Precedents retrieved

- P1: [pg_rewind does not rewind diverging timelines](https://commitfest.postgresql.org/patch/6811/) (cf, Withdrawn, 2026-05-29) - judged not relevant; reason summarised from its thread
- P2: [Data-only mode for pg_rewind](https://commitfest.postgresql.org/patch/2063/) (cf, Returned with feedback, 2019-08-01) - judged not relevant; reason summarised from its thread
- P3: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant; reason summarised from its thread
- P4: [Make pg_rewind not copy useless WALs](https://commitfest.postgresql.org/patch/1302/) (cf, Returned with feedback, 2018-01-31) - used in the review; reason summarised from its thread
- P5: [Add tablespace tap test to pg_rewind](https://commitfest.postgresql.org/patch/2059/) (cf, Returned with feedback, 2019-05-10) - judged not relevant; reason summarised from its thread
- P6: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - used in the review; reason summarised from its thread
- P7: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - used in the review; reason summarised from its thread
- P8: [Fix pg_rewind false positives caused by shutdown-only WAL](https://commitfest.postgresql.org/patch/6041/) (cf, Withdrawn, 2026-05-01) - used in the review; reason summarised from its thread
- P9: [speed up pg_rewind by using copy_file_range() and skipping some fsync](https://commitfest.postgresql.org/patch/2990/) (cf, Returned with feedback, 2021-10-01) - used in the review
- P10: [Trigger failsafe when VACUUM is run in single user mode](https://commitfest.postgresql.org/patch/3467/) (cf, Returned with feedback, 2022-04-07) - judged not relevant
- P11: ["Causal reads" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/487/) (cf, Returned with feedback, 2016-04-06) - judged not relevant
- P12: [Synchronous replay mode for avoiding stale reads on hot standbys](https://commitfest.postgresql.org/patch/1589/) (cf, Withdrawn, 2019-07-01) - judged not relevant
- P13: [Fix Deadlock Issue in Single User Mode When IO Failure Occurs](https://commitfest.postgresql.org/patch/2003/) (cf, Returned with feedback, 2020-02-01) - judged not relevant
- P14: ["synchronous_replay" mode for load balancing reads without stale data](https://commitfest.postgresql.org/patch/951/) (cf, Returned with feedback, 2017-11-29) - judged not relevant
- P15: [Doc: update or remove dead external links.](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=e91cd951b1114a009a0256f211a41d0ec873b2fc) (revert, Partially reverted, 2020-07-10) - judged not relevant
- P16: [Revert "Switch TAP tests of pg_rewind to use a role with minimal permissions"](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=db8db624e826efbe16aab1ae921bae071f98f099) (revert, Reverted, 2019-04-13) - judged not relevant
- P17: [Remove redundant initializations](https://commitfest.postgresql.org/patch/3229/) (cf, Returned with feedback, 2021-10-01) - judged not relevant
- P18: [Remove unused #include's](https://commitfest.postgresql.org/patch/6112/) (cf, Withdrawn, 2025-10-15) - judged not relevant
- P19: [INNER JOIN removals](https://commitfest.postgresql.org/patch/130/) (cf, Returned with feedback, 2015-05-11) - judged not relevant
- P20: [INNER JOIN removals](https://commitfest.postgresql.org/patch/27/) (cf, Rejected, 2015-02-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 14; wall time: 202 s; cost reported by the CLI: $1.46
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 6 from cache, 2 summarised now, 0 failed; quotes: 2 verified, 0 dropped; relevance pass: done, 5 of 20 kept
- Tone check: 96 text field(s) checked, 1 finding(s), 1 sentence(s) rewritten (repair call: done), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:30:30.079Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
