# Design pre-review: pg_rewind does not rewind diverging timelines

> Prepared by an AI harness; treat as a devil's advocate, not a verdict.

- CommitFest: [patch 6768](https://commitfest.postgresql.org/patch/6768/) (Needs review); authors: Mats Kindahl (matski)
- Thread: [pg_rewind does not rewind diverging timelines](https://www.postgresql.org/message-id/flat/CAN305gBeJr8m7ZRW9mH0zakEFR4hDUPDo8fJRKJOHWMORG5_Bg%40mail.gmail.com), started 2026-04-30
- Messages read: 36 of 36
- Patch prose read: v9.0001-pg_rewind-use-UUIDs-to-detect-independent-same-TLI-p.patch, v2.0002-pg_rewind-test-rewind-across-UUID-mismatched-TLI.patch, v2.0001-pg_rewind-use-UUIDs-to-detect-independent-same-TLI-p.patch (commit message, file list and doc hunks only; no code review)

## 1. Verdict

**NEEDS-DESIGN-DISCUSSION** (confidence: medium)

The problem is stated and several independent people confirm it, so levels 1a and 1b pass. The design is still open at level 2. The UUID is only in the history file, so pg_rewind's own archive reads can still pick the other branch's segment (replication-ondisk-1), and the proposal does not say which timeline comparisons must use the UUID (maintenance-fit-1). The back-branch intent conflicts with the format and API change (standards-compat-1).

First failing level, from the base up: **Level 2: architecture and design**. The review stops there; higher levels are not evaluated.

- Level 1a: understandability: passed
- Level 1b: desirability: passed
- Level 2: architecture and design: failed
- Levels 3-5 (implementation): out of scope

Provisional: Only 3 of 36 messages and no code were read. How the patch parses the fourth field next to the free-text reason, and how pg_rewind picks WAL segments after the ancestor search, could not be checked.

## The proposal, restated

**Problem.** Two servers can promote on their own to the same timeline ID, with identical history entries (same TLI, same switchpoint LSN). pg_rewind then treats them as one timeline, so it skips the rewind or picks a too-late common ancestor and leaves diverged data on the target. This affects HA setups with repeated crashes during failover, with or without a shared archive.

**Proposed design.** Each promotion generates a UUIDv7 and writes it as an optional fourth field in each timeline history file entry. pg_rewind compares TLI and UUID both in the common ancestor search and in the same-TLI shortcut, and walks back on a mismatch. A zero UUID (old history file) counts as different from a non-zero UUID; the WAL and the end-of-recovery record are unchanged.

**User-visible surface.**
- Timeline history file format: optional UUID field per entry.
- pg_rewind behaviour: same-TLI shortcut and ancestor search now also compare UUIDs.
- C API: writeTimeLineHistory() gains a UUID argument.
- C API: generate_uuidv7_r() and generate_uuidv7() exported in utils/uuid.h.
- Backend: malformed UUID in a history file causes FATAL at startup, while pg_rewind treats it as unknown.

**What the author claims.**
- The scenario is real, was found by a TLA+ model, and can cause silent divergence between servers.
- The fix is small and touches few places, so the risk is low.
- Even a shared archive cannot guarantee unique TLI assignment, because it is a consensus problem.
- The UUID is an extra check, comparable to page checksums, and does not replace the TLI.
- Clusters with old history files and a missing target history file keep working.

**Thread state.** Japin Li and Surya Poondla approved earlier versions, and Andreas Karlsson and Ants Aasma confirm the problem in practice. Michael Paquier opposes a second timeline identifier as unneeded; Ants Aasma says it does not go far enough without the UUID in WAL page headers, and the patch needs a rebase.

## 2. Prior attempts

### Add timeline to partial WAL segments (related)

- **Link:** [https://commitfest.postgresql.org/patch/1919/](https://commitfest.postgresql.org/patch/1919/)
- **Thread:** [https://www.postgresql.org/message-id/flat/c6433bac-65bc-555e-e318-2077dd153eab@pgmasters.net](https://www.postgresql.org/message-id/flat/c6433bac-65bc-555e-e318-2077dd153eab@pgmasters.net)
- **Date:** decided 2020-09-30; started 2018-12-10
- **Outcome:** Returned with feedback (cf)
- **Who objected:** not recorded in the corpus
- **Why it was not accepted (AI summary of the linked thread; check it there):** The author stopped working on it. After discussion, Michael Paquier agreed with a revised scheme in which pg_receivewal would also use the timeline-suffixed name. The author then marked the patch Waiting on Author because it needed more work, and no new version followed for about 20 months. Robert Haas and Michael Paquier also objected to back-patching the naming change into minor releases, but that did not decide the entry.
- **Reason (quote):** "As nothing has happened here, I have marked the patch as RwF." ([message](https://www.postgresql.org/message-id/20200930071750.GM1996%40paquier.xyz))
- **What is different this time:** P5 renamed archived .partial segments to avoid archive conflicts after several promotions. This proposal adds a per-promotion UUID to history file entries and changes only pg_rewind matching, not WAL file names.


## 3. Objections a reviewer is likely to raise

### 1. Archived WAL segments with the same TLI stay ambiguous to pg_rewind

fixable · level 2, rubric item 7 · WAL / replication / standby, on-disk format / pg_upgrade · `replication-ondisk-1`

S1 and S2 both promote to TLI 2 and share an archive, which then holds 000000020000000000000004 from S2. pg_rewind --restore-target-wal on S1 correctly backs up to TLI 1, then restores that segment as S1's own TLI 2 WAL. It misses S1's changed blocks, and S1 keeps diverged pages with no error.

The UUID lives only in the history file, so nothing ties a WAL segment to the promotion that wrote it. Message #23 raises this and proposes the UUID in XLogLongPageHeaderData. Message #27 calls WAL format changes too intrusive but does not show that pg_rewind's own archive reads are safe. The fix covers ancestor selection and leaves block collection in the same collision.

Thread: [#23 (Ants Aasma, 2026-07-29)](https://www.postgresql.org/message-id/CANwKhkO0jA5-V%2BOhQtPOwscLrq1%2BO9sCZXfuo4MYecjd1r9wag%40mail.gmail.com), [#27 (Mats Kindahl, 2026-08-12)](https://www.postgresql.org/message-id/c9bfdf63-0c04-41c5-bd68-93e62bfa1f21%40gmail.com)

**Question:** When pg_rewind restores target WAL from a shared archive after two same-TLI promotions, what stops it from reading the other branch's segment of the same name?

Counter-check: held. Message #23 raises same-TLI segments in a shared archive, and message #27 answers for recovery but not for pg_rewind's own archive reads. The case needs a missing local segment, so it is narrow, but the thread does not rule it out.

### 2. Second timeline identity checked only by pg_rewind

fixable (first rated fatal; changed in the counter-check) · level 2, rubric item 9 · maintenance burden ("who maintains this in 10 years"), WAL / replication / standby, on-disk format / pg_upgrade · `maintenance-fit-1`

Two nodes promote to TLI 2 and both archive WAL. Recovery or restore_command later replays the wrong node's TLI 2 segments, because only pg_rewind compares the UUID. Operators find out after the restore, when the data has diverged.

The patch makes the UUID part of timeline identity but checks it in one frontend tool. Recovery, the archiver and WAL page headers ignore it. Ants Aasma (message #23) gives the archive case, and the author's reply in message #27 does not close it. Any future code that compares timelines would also need to check the UUID, and nothing fails if it does not.

Thread: [#22 (Michael Paquier, 2026-07-26)](https://www.postgresql.org/message-id/amaZ6bN0pJM75OEs%40paquier.xyz), [#23 (Ants Aasma, 2026-07-29)](https://www.postgresql.org/message-id/CANwKhkO0jA5-V%2BOhQtPOwscLrq1%2BO9sCZXfuo4MYecjd1r9wag%40mail.gmail.com), [#27 (Mats Kindahl, 2026-08-12)](https://www.postgresql.org/message-id/c9bfdf63-0c04-41c5-bd68-93e62bfa1f21%40gmail.com)

**Question:** Which other code paths that compare timelines must honour the UUID, and how would a future change that misses one be caught?

Counter-check: held. The recovery scenario exists today and the patch does not make it worse, so fatal is too strong. The proposal still does not say which timeline comparisons use the UUID and which do not, as message #23 asks.

### 3. Bugfix tag and back-branch intent conflict with format and API changes

fixable · level 2, rubric item 6 · backpatchability, ABI / extension API, on-disk format / pg_upgrade · `standards-compat-1`

A packager ships a minor release with this fix. An HA tool or extension built against the old writeTimeLineHistory() signature breaks on the minor update. In a rolling minor upgrade, patched and unpatched nodes then write history files in two formats in the same cluster and archive.

The CF entry is tagged Bugfix and targets version 18. The patch changes the timeline history file format and the signature of an exported backend function. Message #17 asked whether the signature change is an ABI break if the fix is back-patched, and no later message answers it. A committer would likely ask for a master-only design, or a separate back-branch variant that keeps the old signature and file format.

Thread: [#17 (Zsolt Parragi, 2026-06-08)](https://www.postgresql.org/message-id/CAN4CZFO%2BHVWRWELsu4CuQ%2BFr%3DJ4e%2BjdHM5siRQ3G3vyJ5-Y6Sg%40mail.gmail.com)

**Question:** Is this meant for master only, and if not, how does a back-branch version keep the writeTimeLineHistory() ABI and the history file format stable within a major release?

Counter-check: held. The CF entry says Bugfix with target 18, and message #17's ABI question on writeTimeLineHistory() has no answer in the thread. The patch prose has no back-branch plan, so the question stands.

### 4. History file format change has no stated contract for external parsers

fixable · level 2, rubric item 12 · on-disk format / pg_upgrade, WAL / replication / standby, maintenance burden ("who maintains this in 10 years") · `standards-compat-2`

A backup or HA tool reads a new history file, from the archive or through the TIMELINE_HISTORY replication command, and splits each line on tabs. It shows the UUID as the switch reason, or rejects a line with an unexpected field count. The operator finds out during a failover or a restore.

History files are read by more than the server and pg_rewind: they travel through archives and replication to tools the project does not control. The proposal adds an optional fourth field, but the patch contains no documentation hunk. It does not say where the field sits relative to the free-text reason, or how an older server or older tool should treat it. [P22] shows that a change in field count broke replication utilities across versions, and that commit was reverted.

**Precedents.**
- [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (Reverted, 2014-08-19)

Thread: [#17 (Zsolt Parragi, 2026-06-08)](https://www.postgresql.org/message-id/CAN4CZFO%2BHVWRWELsu4CuQ%2BFr%3DJ4e%2BjdHM5siRQ3G3vyJ5-Y6Sg%40mail.gmail.com), [#21 (Tatsuya Kawata, 2026-07-26)](https://www.postgresql.org/message-id/CAHza6qczQXOwEyqNX2rq5UNo2XQb_bWY33F%3D9sNXzVPr6DztMA%40mail.gmail.com)

**Question:** What is the exact new line layout, where is it documented, and how do older servers and existing tools that parse history files read a line with the UUID field?

Counter-check: held. The patch prose shows no documentation of the new line layout and no statement on older readers. P22 concerns the IDENTIFY_SYSTEM result, not history files, so it should not be used as support.

## 4. Touch points

| Touch point | Hit | Why |
|---|---|---|
| on-disk format / pg_upgrade | **yes** | Each timeline history file entry gains an optional fourth field with a UUID. |
| WAL / replication / standby | **yes** | Promotion writes the UUID, and pg_rewind uses it to match timelines between source and target. |
| wire protocol | no |  |
| catalog changes | no |  |
| ABI / extension API | **yes** | writeTimeLineHistory() gains a UUID argument, and utils/uuid.h exports generate_uuidv7_r() and generate_uuidv7(). |
| SQL-standard conflict or non-standard syntax | no |  |
| new GUC | no |  |
| planner behaviour change | no |  |
| pg_dump / restore round-trip | no |  |
| psql UX conventions | no |  |
| security / privileges | no |  |
| backpatchability | **yes** | The CF entry is tagged Bugfix with target 18, but the patch changes a file format and an exported function signature. |
| maintenance burden ("who maintains this in 10 years") | **yes** | The UUID becomes a second timeline identity that only pg_rewind checks; later timeline code must know about it. |
| performance on the hot path | no |  |
| doesn't fit what Postgres is about (extension / external tool / already solved) | **yes** | Message #22 argues that the archive design should guarantee unique TLIs, so a second identifier is unneeded. |

## 5. Questions the author must answer before anyone reviews code

1. When pg_rewind restores target WAL from a shared archive after two same-TLI promotions, what stops it from reading the other branch's segment of the same name?
2. Which code paths that compare timelines must use the UUID, which may ignore it, and how would a future change that misses one be caught?
3. Is the patch meant for master only, and if not, how does a back-branch version keep the writeTimeLineHistory() signature and the history file format unchanged?
4. What is the exact new history file line layout, where is it documented, and how do older servers and external tools read a line with the UUID field?
5. Can a zero UUID compared with a non-zero UUID force a rewind that is not needed when one node was promoted before the upgrade and another after it?

## Committability

"Will not get committed" is not the same as "should be rejected". The verdict above is about the merits (levels 1-2) only; this section is about who would sign for it, and it never changes the verdict.

- **Owner:** qualified committer engaged. Michael Paquier takes part in the thread but opposes a second timeline identifier (message #22). The owner must know timeline history handling, promotion in StartupXLOG() and pg_rewind.
- **Liability:** The owner signs for the history file format for all later releases, for pg_rewind correctness in mixed old and new clusters, and for any back-branch variant.
- **Author follow-up:** The author posted nine versions up to message #33 and answered review comments. The CF entry needs a rebase since 2026-10-22.

**Not now / show demand:** does not apply.

## Priority

Whether it is worth the reviewers' time now. This is apart from the verdict and never changes it.

- **Recommended action:** fix design first
- **Audience:** a sizeable minority
- **Maintenance burden:** medium
- **Review cost:** medium
- **Why:** HA users who run pg_rewind after failover are affected, and messages #19, #23 and #24 report the problem in practice. The patch is about 44 KiB across xlog.c, timeline.c and pg_rewind, and the thread has not agreed on whether a UUID in the history file alone is the right scope.

## Appendix

### Refuted in the counter-check

- `replication-ondisk-2` Startup FATAL on a history file field that recovery never uses. Refuted (already handled): Message #3 notes that the existing syntax-error paths in readTimeLineHistory() use FATAL on purpose, and the author matched them in message #4. The patch follows existing practice for a malformed history file.
- `replication-ondisk-3` Bugfix entry changes the history file format and an exported function. Refuted (not specific): Duplicate of standards-compat-1. P5 was returned because the author stopped working on it, so it adds no support.
- `maintenance-fit-2` Committer in the area opposes the mechanism. Refuted (already handled): Messages #23, #24 and #26 answer the premise that an archive can guarantee unique TLIs, and message #31 keeps the UUID as a last-resort check beside a coordinated TLI. Ownership belongs in the committability field, which the rubric keeps apart from the verdict.
- `maintenance-fit-3` Format and API change filed as a back-patchable bugfix. Refuted (not specific): Duplicate of standards-compat-1 for the back-patch and ABI part, and of standards-compat-2 for external parsers.

### Review angles

- prior-attempts: 0 objection(s). No listed precedent was rejected or reverted for a reason that applies to this proposal. P5 died because work stopped, and its back-patching concern appears only in an AI summary. In message #22, Michael Paquier cites the project's position that unique TLI assignment depends on the archive, and message #31 links a built-in consensus thread, but neither reports an earlier rejection of this idea.
- standards-compat: 2 objection(s). The material contains no new SQL syntax, GUC, psql change or pg_dump object, so these points need no objection.
- replication-ondisk: 3 objection(s). Objections were found from this angle.
- security: 0 objection(s). The proposal has no privilege or exposure aspect. The UUID is written only into timeline history files that the server or pg_rewind creates, and only the server and pg_rewind read it. No SQL function, role, view or GUC is added, and the history file stays as readable as it is today. A malformed UUID causes FATAL at startup, which matches the existing FATAL paths for malformed history files, and the archive is already trusted input. The use-after-free in the error path (messages #17, #21) was a code defect and was fixed in the thread.
- maintenance-fit: 3 objection(s). Hot-path cost is not a concern here. The UUID is written only at promotion and read only by pg_rewind and at startup.
- already-exists: 0 objection(s). Nothing in Postgres today does this job. The single shared archive is the documented way to keep TLI unique (message #22), but messages #23 and #26 show that a check through restore_command followed by asynchronous archiving cannot make TLI selection atomic. pg_rewind has no other way to tell two same-TLI promotions apart. The DCS-supplied TLI or TLI-allocation hook from messages #31 and #32 does not exist yet, and it would not cover promotions that no coordinator manages. Points not checked without the code: how the patch interacts with tools that parse history files, and whether pg_rewind could get the same result from data it already reads.
- demand: 0 objection(s). The thread shows real demand from several independent people. Japin Li hit the issue in practice (message #19), Andreas Karlsson met it while debugging a Kubernetes operator (message #24), and Ants Aasma reports correlated failover failures in fleets (message #23). The cost falls only on promotion and on pg_rewind, so there is no demand-angle objection. Michael Paquier's position in message #22 is a design question for other angles: whether a single archive should already guarantee a unique TLI.

### Precedents retrieved

- P1: [Archive current timeline history file on archiver startup if needed](https://commitfest.postgresql.org/patch/4502/) (cf, Returned with feedback, 2024-02-01) - judged not relevant; reason summarised from its thread
- P2: [Logical Decoding with timelines](https://commitfest.postgresql.org/patch/47/) (cf, Returned with feedback, 2015-05-03) - judged not relevant; reason summarised from its thread
- P3: [Revert timeline following in replication slots](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c1543a81a7a89207b6c45b8f3f7f07b1148fcc6e) (revert, Reverted, 2016-05-04) - judged not relevant
- P4: [get current and recovery timeline](https://commitfest.postgresql.org/patch/2225/) (cf, Returned with feedback, 2020-01-31) - judged not relevant; reason summarised from its thread
- P5: [Add timeline to partial WAL segments](https://commitfest.postgresql.org/patch/1919/) (cf, Returned with feedback, 2020-09-30) - used in the review; reason summarised from its thread
- P6: [\[BUG\] Panic due to incorrect missingContrecPtr after promotion](https://commitfest.postgresql.org/patch/3675/) (cf, Withdrawn, 2022-09-21) - judged not relevant; reason summarised from its thread
- P7: [Logical decoding timeline following take II](https://commitfest.postgresql.org/patch/779/) (cf, Returned with feedback, 2016-11-29) - judged not relevant
- P8: [Restrict pg_rewind to specified/needed directories](https://commitfest.postgresql.org/patch/1343/) (cf, Returned with feedback, 2018-02-01) - judged not relevant; reason summarised from its thread
- P9: [Avoid Copying WAL Segments Before Divergence to Speed Up pg_rewind](https://commitfest.postgresql.org/patch/3800/) (cf, Returned with feedback, 2022-11-30) - judged not relevant; reason summarised from its thread
- P10: [Allow PostgreSQL to perform promote/pause/shutdown operations at end-of-the-wal when the recovery does not reach the recovery target](https://commitfest.postgresql.org/patch/847/) (cf, Returned with feedback, 2017-03-27) - judged not relevant; reason summarised from its thread
- P11: [Data-only mode for pg_rewind](https://commitfest.postgresql.org/patch/2063/) (cf, Returned with feedback, 2019-08-01) - judged not relevant
- P12: [Make pg_rewind not copy useless WALs](https://commitfest.postgresql.org/patch/1302/) (cf, Returned with feedback, 2018-01-31) - judged not relevant
- P13: [pg_rewind with cascade standby doesn't work well](https://commitfest.postgresql.org/patch/4559/) (cf, Returned with feedback, 2024-10-07) - used in the review
- P14: [Fix base backup checksum verification for random or zero page headers](https://commitfest.postgresql.org/patch/2308/) (cf, Returned with feedback, 2020-07-30) - judged not relevant
- P15: [Add tablespace tap test to pg_rewind](https://commitfest.postgresql.org/patch/2059/) (cf, Returned with feedback, 2019-05-10) - judged not relevant
- P16: [Allow providing restore_command as a command line option to pg_rewind](https://commitfest.postgresql.org/patch/3213/) (cf, Rejected, 2022-04-07) - judged not relevant
- P17: [array_random](https://commitfest.postgresql.org/patch/5881/) (cf, Withdrawn, 2026-01-06) - judged not relevant
- P18: [Identify huge pages accessibility using madvise](https://commitfest.postgresql.org/patch/4937/) (cf, Withdrawn, 2024-11-12) - judged not relevant
- P19: [Fix checksum verification in base backups for zero page headers](https://commitfest.postgresql.org/patch/2719/) (cf, Returned with feedback, 2021-09-13) - judged not relevant
- P20: [Creating backup history files for backups taken from standbys](https://commitfest.postgresql.org/patch/1231/) (cf, Returned with feedback, 2018-03-05) - judged not relevant
- P21: [Review of arc4random for strong randomness](https://commitfest.postgresql.org/patch/1382/) (cf, Rejected, 2018-01-12) - judged not relevant
- P22: [Revert "Fix bug in checking of IDENTIFY_SYSTEM result."](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=c4762886539b06da682045e19ba73dcee3789638) (revert, Reverted, 2014-08-19) - used in the review
- P23: [Revert recent changes with durable_rename_excl()](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commit;h=55b56865115eccd6449e79d6f06fe49d6ba3b792) (revert, Reverted, 2022-04-28) - used in the review
- P24: [Extend pgbench partitioning to pgbench_history](https://commitfest.postgresql.org/patch/4679/) (cf, Returned with feedback, 2024-05-14) - judged not relevant

### Run

- Backend: claude CLI on the subscription login (no API key); models: review angles opus, counter-check claude-mythos-5-2, synthesis claude-mythos-5-2; no tools given to any of them
- LLM calls: 11; wall time: 149 s; cost reported by the CLI: $3.07
- Precedent enrichment: 8 thread(s) lacked a recorded reason: 8 from cache, 0 summarised now, 0 failed; quotes: 0 verified, 0 dropped; relevance pass: done, 4 of 24 kept
- Tone check: 90 text field(s) checked, 0 finding(s), 0 sentence(s) rewritten (repair call: not needed), 0 left
- Sanitizer: 0 link(s) removed, 0 unknown precedent id(s) dropped, 0 bad message reference(s) dropped, 0 unverifiable quote(s) dropped
- Generated: 2026-11-12T03:02:52.144Z by pgroast/0.1

---

Prepared by an AI harness; treat as a devil's advocate, not a verdict.
